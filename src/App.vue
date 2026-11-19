<script setup>
import { computed, ref } from 'vue'
import MovieCard from './components/MovieCard.vue'

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:5000'

const username = ref('')
const loadedUsername = ref(null)
const watchlistCount = ref(null)
const movies = ref([])
const selectedGenre = ref('')
const selectedDuration = ref('')
const movie = ref(null)
const loading = ref(null) // 'watchlist' | 'pick' | null
const error = ref('')
const usernameError = ref('')
const slowRequest = ref(false)

const durationRanges = [
  { value: '', label: 'Cualquier duración', matches: () => true },
  { value: 'short', label: 'Menos de 90 min', matches: (minutes) => minutes < 90 },
  { value: 'medium', label: '90 a 120 min', matches: (minutes) => minutes >= 90 && minutes <= 120 },
  { value: 'long', label: 'Más de 120 min', matches: (minutes) => minutes > 120 },
]

const availableGenres = computed(() =>
  [...new Set(movies.value.flatMap((item) => item.genres || []))].sort((a, b) => a.localeCompare(b)),
)

const hasFilters = computed(() => Boolean(selectedGenre.value || selectedDuration.value))

const filteredMovies = computed(() => {
  const range = durationRanges.find((item) => item.value === selectedDuration.value)

  return movies.value.filter((item) => {
    const matchesGenre = !selectedGenre.value || item.genres?.includes(selectedGenre.value)
    const matchesDuration = !range || (item.durationMinutes != null && range.matches(item.durationMinutes))
    return matchesGenre && matchesDuration
  })
})

let activeController = null
let wakingTimer = null

function cancelPending() {
  if (activeController) activeController.abort()
  if (wakingTimer) clearTimeout(wakingTimer)
  activeController = null
  wakingTimer = null
}

function begin(label) {
  cancelPending()
  activeController = new AbortController()
  loading.value = label
  error.value = ''
  usernameError.value = ''
  movie.value = null
  slowRequest.value = false

  wakingTimer = setTimeout(() => {
    if (loading.value) slowRequest.value = true
  }, 5000)

  return activeController.signal
}

function finish(signal) {
  if (activeController?.signal !== signal) return
  if (wakingTimer) clearTimeout(wakingTimer)
  wakingTimer = null
  activeController = null
  loading.value = null
  slowRequest.value = false
}

function messageFor(status, detail) {
  if (typeof detail === 'string' && detail.length > 0 && detail.length < 120) return detail

  switch (status) {
    case 400:
      return 'Revisá el nombre de usuario de Letterboxd.'
    case 404:
      return 'No encontramos ese usuario, o su watchlist es privada o está vacía.'
    case 429:
    case 502:
      return 'Letterboxd no está respondiendo bien. Probá de nuevo en un momento.'
    case 504:
      return 'La solicitud tardó demasiado. Probá de nuevo.'
    default:
      return 'No pudimos conectar con el servidor.'
  }
}

async function request(path, label) {
  const name = username.value.trim()
  if (!name) {
    usernameError.value = 'Escribí tu usuario de Letterboxd para comenzar.'
    return null
  }

  const signal = begin(label)

  try {
    const response = await fetch(
      `${API_URL}/api/watchlist/${encodeURIComponent(name)}${path}`,
      { signal, headers: { Accept: 'application/json' } },
    )

    let data = null
    try {
      data = await response.json()
    } catch {
    }

    if (!response.ok) {
      error.value = messageFor(response.status, data?.detail)
      return null
    }

    return data
  } catch (err) {
    if (err?.name !== 'AbortError') {
      error.value = messageFor(0)
    }
    return null
  } finally {
    finish(signal)
  }
}

async function loadWatchlist() {
  const name = username.value.trim().toLowerCase()
  if (loadedUsername.value === name && watchlistCount.value !== null) return

  const data = await request('', 'watchlist')
  if (data) {
    movies.value = data.movies ?? []
    watchlistCount.value = data.totalMovies ?? movies.value.length
    loadedUsername.value = name
  }
}

function pickRandom() {
  const pool = filteredMovies.value
  if (!pool.length) {
    error.value = 'No hay películas con esos filtros.'
    movie.value = null
    return
  }

  error.value = ''
  movie.value = pool[Math.floor(Math.random() * pool.length)]
}

function onFilterChange() {
  movie.value = null
  error.value = ''
}

function clearUsername() {
  username.value = ''
  onUsernameInput()
}

function onUsernameInput() {
  if (loading.value) cancelPending()
  loading.value = null
  loadedUsername.value = null
  watchlistCount.value = null
  movies.value = []
  selectedGenre.value = ''
  selectedDuration.value = ''
  movie.value = null
  error.value = ''
  usernameError.value = ''
}
</script>

<template>
  <div class="app">
    <!-- Barra Superior / Header Letterboxd -->
    <header class="header">
      <div class="brand-wrapper">
        <div class="brand">
          <span class="letterboxd-dots" aria-hidden="true">
            <span class="dot dot-green"></span>
            <span class="dot dot-orange"></span>
            <span class="dot dot-blue"></span>
          </span>
          <span class="brand-name">letterboxd movie picker</span>
        </div>
      </div>
    </header>

    <!-- Contenedor Principal -->
    <main class="main">
      <h1 class="hero-title">¿Qué película miramos hoy?</h1>

      <!-- Panel de Búsqueda -->
      <form class="search-panel" @submit.prevent="loadWatchlist">
        <label class="sr-only" for="username">Usuario de Letterboxd</label>
        <div class="search-input-wrapper">
          <svg class="search-icon" viewBox="0 0 24 24" aria-hidden="true">
            <circle cx="11" cy="11" r="7" />
            <path d="m20 20-3.5-3.5" />
          </svg>
          <input
            id="username"
            v-model="username"
            type="text"
            placeholder="Ingresá tu usuario de Letterboxd"
            autocomplete="username"
            spellcheck="false"
            :aria-invalid="Boolean(usernameError)"
            @input="onUsernameInput"
          />
          <button
            v-if="username"
            type="button"
            class="input-clear-btn"
            title="Borrar texto"
            aria-label="Borrar usuario"
            @click="clearUsername"
          >
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none">
              <line x1="18" y1="6" x2="6" y2="18" />
              <line x1="6" y1="6" x2="18" y2="18" />
            </svg>
          </button>
        </div>
      </form>

      <!-- Controles: botones y filtros en la misma fila -->
      <div class="controls">
        <!-- Botones de Acción -->
        <div class="actions">
          <button
            type="button"
            class="button button-secondary"
            :disabled="loading !== null"
            @click="loadWatchlist"
          >
            <span v-if="loading === 'watchlist'" class="spinner" aria-hidden="true"></span>
            <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
              <polyline points="7 10 12 15 17 10" />
              <line x1="12" y1="15" x2="12" y2="3" />
            </svg>
            <span>{{ loading === 'watchlist' ? 'Cargando…' : 'Cargar watchlist' }}</span>
          </button>

          <button
            v-if="watchlistCount !== null"
            type="button"
            class="button button-primary"
            :disabled="loading !== null"
            @click="pickRandom"
          >
            <span v-if="loading === 'pick'" class="spinner" aria-hidden="true"></span>
            <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2">
              <polyline points="16 3 21 3 21 8" />
              <line x1="4" y1="20" x2="21" y2="3" />
              <polyline points="21 16 21 21 16 21" />
              <line x1="15" y1="15" x2="21" y2="21" />
              <line x1="4" y1="4" x2="9" y2="9" />
            </svg>
            <span>{{ loading === 'pick' ? 'Eligiendo…' : 'Elegir al azar' }}</span>
          </button>
        </div>

        <!-- Filtros -->
        <div v-if="watchlistCount !== null" class="filters">
          <div class="filter-field">
            <select v-model="selectedGenre" aria-label="Género" @change="onFilterChange">
              <option value="">Todos los géneros</option>
              <option v-for="genre in availableGenres" :key="genre" :value="genre">
                {{ genre }}
              </option>
            </select>
          </div>

          <div class="filter-field">
            <select v-model="selectedDuration" aria-label="Duración" @change="onFilterChange">
              <option v-for="range in durationRanges" :key="range.value" :value="range.value">
                {{ range.label }}
              </option>
            </select>
          </div>
        </div>
      </div>

      <!-- Estados y Notificaciones -->
      <div class="status" aria-live="polite">
        <!-- Error -->
        <div v-if="error || usernameError" class="error-badge">
          <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none">
            <circle cx="12" cy="12" r="10" />
            <line x1="12" y1="8" x2="12" y2="12" />
            <line x1="12" y1="16" x2="12.01" y2="16" />
          </svg>
          <span>{{ error || usernameError }}</span>
        </div>

        <!-- Conteo de Películas -->
        <p v-else-if="watchlistCount !== null" class="count-badge">
          <template v-if="hasFilters">
            {{ filteredMovies.length }} de {{ watchlistCount }} películas
          </template>
          <template v-else>{{ watchlistCount }} películas en tu watchlist</template>
        </p>

        <!-- Servidor Despertando -->
        <p v-if="slowRequest" class="waking">
          La carga está tardando un poco más de lo esperado…
        </p>
      </div>

      <!-- Componente de Tarjeta de Película -->
      <MovieCard v-if="movie" :movie="movie" />

      <!-- Botón para Repetir Selección -->
      <div v-if="movie" class="again">
        <button
          type="button"
          class="button button-secondary"
          :disabled="loading !== null"
          @click="pickRandom"
        >
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M23 4v6h-6" />
            <path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10" />
          </svg>
          <span>Elegir otra</span>
        </button>
      </div>
    </main>
  </div>
</template>
