<script setup>
import { nextTick, onMounted, ref, watch } from 'vue'

const props = defineProps({
  movie: {
    type: Object,
    required: true,
  },
})

const posterRef = ref(null)
const imageLoaded = ref(false)
const imageFailed = ref(false)

async function syncPosterState() {
  await nextTick()
  const image = posterRef.value
  if (!image?.complete) return

  if (image.naturalWidth > 0) imageLoaded.value = true
  else imageFailed.value = true
}

watch(
  () => props.movie.url,
  () => {
    imageLoaded.value = false
    imageFailed.value = false
    syncPosterState()
  },
)

onMounted(syncPosterState)
</script>

<template>
  <article class="movie-card">
    <!-- Contenedor del Póster 2:3 -->
    <div class="poster-container">
      <div class="poster">
        <div
          v-if="movie.posterUrl && !imageLoaded && !imageFailed"
          class="poster-skeleton"
          aria-hidden="true"
        ></div>
        <img
          v-if="movie.posterUrl && !imageFailed"
          :key="movie.url"
          ref="posterRef"
          class="poster-image"
          :class="{ 'is-loaded': imageLoaded }"
          :src="movie.posterUrl"
          :alt="`Póster oficial de ${movie.title}`"
          loading="eager"
          decoding="async"
          fetchpriority="high"
          @load="imageLoaded = true"
          @error="imageFailed = true"
        />
        <div
          v-else
          class="poster-placeholder"
          role="img"
          :aria-label="`Sin póster para ${movie.title}`"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <rect x="3" y="3" width="18" height="18" rx="2" />
            <path d="M7 3v18M17 3v18M3 8h4M3 16h4M17 8h4M17 16h4M7 12h10" />
          </svg>
          <span>Sin póster</span>
        </div>
      </div>
    </div>

    <!-- Información y Metadatos de la Película -->
    <div class="movie-info">
      <div class="movie-header-group">
        <h2 class="movie-title">{{ movie.title }}</h2>

        <!-- Metadatos: Año y Duración -->
        <div class="movie-meta-row">
          <span v-if="movie.year" class="meta-year">{{ movie.year }}</span>

          <span v-if="movie.durationMinutes" class="meta-duration">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <polyline points="12 6 12 12 16 14" />
            </svg>
            {{ movie.durationMinutes }} min
          </span>

          <span v-if="movie.director" class="meta-duration">
            Dir. <strong>{{ movie.director }}</strong>
          </span>
        </div>

        <!-- Géneros -->
        <div v-if="movie.genres?.length" class="meta-genres">
          <span
            v-for="genre in movie.genres"
            :key="genre"
            class="genre-tag"
          >
            {{ genre }}
          </span>
        </div>

        <!-- Sinopsis si está disponible -->
        <p v-if="movie.overview || movie.synopsis" class="movie-overview">
          {{ movie.overview || movie.synopsis }}
        </p>
      </div>

      <!-- Barra Inferior: promedio real traído desde la ficha de Letterboxd -->
      <div class="movie-interactive-bar">
        <div
          v-if="movie.rating"
          class="rating-stars"
          :title="`Promedio: ${movie.rating.toFixed(1)} / 5`"
        >
          <svg
            v-for="star in 5"
            :key="star"
            class="star-icon"
            :class="{ 'star-empty': star > Math.round(movie.rating) }"
            viewBox="0 0 24 24"
          >
            <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2" />
          </svg>
          <span class="rating-score">{{ movie.rating.toFixed(1) }}</span>
        </div>

        <!-- Enlace Oficial a Letterboxd -->
        <a
          :href="movie.url"
          target="_blank"
          rel="noopener noreferrer"
          class="movie-link"
        >
          <span>Ver en Letterboxd</span>
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2">
            <line x1="7" y1="17" x2="17" y2="7" />
            <polyline points="7 7 17 7 17 17" />
          </svg>
        </a>
      </div>
    </div>
  </article>
</template>
