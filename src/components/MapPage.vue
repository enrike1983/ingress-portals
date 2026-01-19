<template>
  <div style="padding: 16px; background: #f8f8f8; z-index: 1000; position: absolute; width: 100vw;">
    <form @submit.prevent="applyFilters" style="display: flex; gap: 12px; align-items: center; flex-wrap: wrap;">
      <label>
        Da:
        <input type="date" v-model="dateFrom" />
        <select v-model="hourFrom">
          <option v-for="h in 24" :key="h-1" :value="h-1">{{ (h-1).toString().padStart(2, '0') }}:00</option>
        </select>
      </label>
      <label>
        A:
        <input type="date" v-model="dateTo" />
        <select v-model="hourTo">
          <option v-for="h in 24" :key="h-1" :value="h-1">{{ (h-1).toString().padStart(2, '0') }}:00</option>
        </select>
      </label>
      <label style="position:relative;">
        Utente:
        <input
          type="text"
          v-model="playerFilter"
          placeholder="Nome utente"
          @input="showUserSuggestions = true"
          @focus="showUserSuggestions = true"
          @blur="hideSuggestions"
          autocomplete="off"
        />
        <ul v-if="showUserSuggestions && filteredUserSuggestions.length"
            style="position:absolute; background:white; border:1px solid #ccc; z-index:1001; list-style:none; margin:0; padding:0; width:100%;">
          <li
            v-for="user in filteredUserSuggestions"
            :key="user"
            @mousedown.prevent="selectUser(user)"
            style="padding:4px 8px; cursor:pointer;"
          >{{ user }}</li>
        </ul>
      </label>
      <button type="submit">Filtra</button>
      <button type="button" @click="resetFilters">Reset</button>
    </form>
  </div>
  <div id="map" style="height: 100vh; width: 100vw"></div>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const center = [46.5, 11.3]
const allMarkers = ref([])
const filteredMarkers = ref([])

const dateFrom = ref('')
const dateTo = ref('')
const hourFrom = ref(0)
const hourTo = ref(23)
const playerFilter = ref('')

const userList = ref([])
const showUserSuggestions = ref(false)

const filteredUserSuggestions = computed(() => {
  if (!playerFilter.value) return userList.value
  return userList.value.filter(u =>
    u.toLowerCase().includes(playerFilter.value.toLowerCase())
  )
})

let map
let leafletMarkers = []

function selectUser(user) {
  playerFilter.value = user
  showUserSuggestions.value = false
}

function hideSuggestions() {
  // Delay per permettere il click su suggestion
  setTimeout(() => { showUserSuggestions.value = false }, 150)
}

async function fetchMarkers(params = {}) {
  const username = 'ingress'
  const password = 'Qu3enai5'
  const basicAuth = 'Basic ' + btoa(`${username}:${password}`)

  // Costruisci la query string dai parametri
  const query = new URLSearchParams()
  if (params.timestamp_from) query.append('timestamp_from', params.timestamp_from)
  if (params.timestamp_to) query.append('timestamp_to', params.timestamp_to)
  if (params.player_name) query.append('player_name', params.player_name)

  const url = 'https://ingress.mrosati.it/plexts/from-db' + (query.toString() ? `?${query.toString()}` : '')

  const res = await fetch(url, {
    headers: { 'Authorization': basicAuth }
  })
  const data = await res.json()
  allMarkers.value = data.plexts || []
  filteredMarkers.value = allMarkers.value

  // Estrai lista utenti unici per autocomplete
  const users = new Set()
  allMarkers.value.forEach(m => {
    if (m.player_name) users.add(m.player_name)
  })
  userList.value = Array.from(users).sort((a, b) => a.localeCompare(b))
}

function clearMapMarkers() {
  leafletMarkers.forEach(m => map.removeLayer(m))
  leafletMarkers = []
}

function renderMarkers() {
  clearMapMarkers()
  filteredMarkers.value.forEach(marker => {
    if (
      marker.coordinates &&
      marker.coordinates.coordinates &&
      Array.isArray(marker.coordinates.coordinates) &&
      marker.coordinates.coordinates.length === 2
    ) {
      // Colore in base al team
      let color = 'red'
      if (marker.team === 'RESISTANCE') color = 'blue'
      else if (marker.team === 'ENLIGHTENED') color = 'green'

      const icon = L.icon({
        iconUrl: `https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-${color}.png`,
        iconSize: [25, 41],
        iconAnchor: [12, 41],
        popupAnchor: [1, -34],
        shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
        shadowSize: [41, 41]
      })

      const leafletMarker = L.marker(
        [marker.coordinates.coordinates[1], marker.coordinates.coordinates[0]],
        { icon }
      )
        .addTo(map)
        .bindPopup(`
          <div>
            <strong>${marker.portal_name || ''}</strong><br />
            ${marker.text || ''}<br />
            <span>Player: <b>${marker.player_name || ''}</b></span><br />
            <small>${marker.timestamp_formatted || ''}</small>
          </div>
        `)
      leafletMarkers.push(leafletMarker)
    }
  })
}

function applyFilters() {
  let params = {}
  if (dateFrom.value) {
    const from = new Date(dateFrom.value)
    from.setHours(hourFrom.value, 0, 0, 0)
    params.timestamp_from = from.getTime()
  }
  if (dateTo.value) {
    const to = new Date(dateTo.value)
    to.setHours(hourTo.value, 59, 59, 999)
    params.timestamp_to = to.getTime()
  }
  if (playerFilter.value) {
    params.player_name = playerFilter.value
  }
  fetchMarkers(params).then(() => {
    renderMarkers()
  })
}

function resetFilters() {
  dateFrom.value = ''
  dateTo.value = ''
  hourFrom.value = 0
  hourTo.value = 23
  playerFilter.value = ''
  fetchMarkers().then(() => {
    renderMarkers()
  })
}

onMounted(async () => {
  await fetchMarkers()
  map = L.map('map').setView(center, 6)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map)
  renderMarkers()
})

// Aggiorna marker quando cambia la lista filtrata
watch(filteredMarkers, () => {
  renderMarkers()
})
</script>
