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
    <div style="margin-top: 8px; display: flex; gap: 8px;">
      <button type="button" @click="startReplay" :disabled="isReplaying || sortedReplayMarkers.length === 0">▶️ Play</button>
      <button type="button" @click="pauseReplay" :disabled="!isReplaying">⏸️ Pausa</button>
      <button type="button" @click="resetReplay" :disabled="!isReplayActive">🔄 Reset</button>
      <span v-if="isReplayActive" style="margin-left: 12px;">Replay: {{ replayCurrentTimeFormatted }}</span>
    </div>
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

// Inizializzazione filtri "da" e "a" predefiniti (un'ora fa -> ora attuale)
const now = new Date()
const unOraFa = new Date(now.getTime() - 60 * 60 * 1000)
dateFrom.value = unOraFa.toISOString().slice(0, 10)
hourFrom.value = unOraFa.getHours()
dateTo.value = now.toISOString().slice(0, 10)
hourTo.value = now.getHours()

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

// Stato per la riproduzione
const isReplaying = ref(false)
const isReplayActive = ref(false)
const replayIndex = ref(0)
const replayTimer = ref(null)
const replayCurrentTime = ref(null)

// Markers ordinati per timestamp
const sortedReplayMarkers = computed(() => {
  return [...filteredMarkers.value].sort((a, b) => a.timestamp - b.timestamp)
})

const replayCurrentTimeFormatted = computed(() => {
  if (!replayCurrentTime.value) return ''
  const d = new Date(replayCurrentTime.value)
  return d.toLocaleString()
})

function selectUser(user) {
  playerFilter.value = user
  showUserSuggestions.value = false
}

function hideSuggestions() {
  setTimeout(() => { showUserSuggestions.value = false }, 150)
}

async function fetchMarkers(params = {}) {
  const username = import.meta.env.VITE_API_USERNAME
  const password = import.meta.env.VITE_API_PASSWORD
  const basicAuth = 'Basic ' + btoa(`${username}:${password}`)

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
  // Reimposta i filtri all'intervallo predefinito (un'ora fa -> ora attuale)
  const now = new Date()
  const unOraFa = new Date(now.getTime() - 60 * 60 * 1000)
  dateFrom.value = unOraFa.toISOString().slice(0, 10)
  hourFrom.value = unOraFa.getHours()
  dateTo.value = now.toISOString().slice(0, 10)
  hourTo.value = now.getHours()
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

watch(filteredMarkers, () => {
  renderMarkers()
})

// Riproduzione replay
function startReplay() {
  if (!sortedReplayMarkers.value.length) return
  isReplaying.value = true
  isReplayActive.value = true
  replayIndex.value = 0
  replayCurrentTime.value = null
  clearMapMarkers()
  // Inizializza array per i marker già mostrati
  shownReplayMarkers.value = []
  replayStep()
}

// Array reattivo per tenere traccia dei marker già mostrati
const shownReplayMarkers = ref([])

function replayStep() {
  if (!isReplaying.value) return
  if (replayIndex.value >= sortedReplayMarkers.value.length) {
    isReplaying.value = false
    return
  }
  const currentTs = sortedReplayMarkers.value[replayIndex.value].timestamp
  replayCurrentTime.value = currentTs
  const sameTsMarkers = []
  while (
      replayIndex.value < sortedReplayMarkers.value.length &&
      sortedReplayMarkers.value[replayIndex.value].timestamp === currentTs
      ) {
    sameTsMarkers.push(sortedReplayMarkers.value[replayIndex.value])
    replayIndex.value++
  }
  // Aggiungi i nuovi marker a quelli già mostrati
  shownReplayMarkers.value = shownReplayMarkers.value.concat(sameTsMarkers)
  showReplayMarkers(sameTsMarkers)
  replayTimer.value = setTimeout(replayStep, 1000)
}

function showReplayMarkers(markers) {
  // NON cancellare i marker già presenti!
  const latlngs = []
  markers.forEach(marker => {
    if (
        marker.coordinates &&
        marker.coordinates.coordinates &&
        Array.isArray(marker.coordinates.coordinates) &&
        marker.coordinates.coordinates.length === 2
    ) {
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

      const latlng = [marker.coordinates.coordinates[1], marker.coordinates.coordinates[0]]
      latlngs.push(latlng)

      const leafletMarker = L.marker(
          latlng,
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
  if (latlngs.length === 1) {
    map.setView(latlngs[0], 15)
  } else if (latlngs.length > 1) {
    map.fitBounds(latlngs, { padding: [50, 50] })
  }
}

function pauseReplay() {
  isReplaying.value = false
  if (replayTimer.value) clearTimeout(replayTimer.value)
}

function resetReplay() {
  pauseReplay()
  isReplayActive.value = false
  replayIndex.value = 0
  replayCurrentTime.value = null
  clearMapMarkers()
  renderMarkers()
}
</script>
