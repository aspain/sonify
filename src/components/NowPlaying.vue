<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :key="player.trackAlbum.image"
          :src="player.trackAlbum.image"
          :crossorigin="player.trackAlbum.image.includes(':1400/') ? null : 'anonymous'"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

export default {
  name: 'NowPlaying',

  props: {
    player: {
      type: Object,
      required: true,
      default: () => ({
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackTitle: ''
      })
    }
  },

  computed: {
    getTrackArtists() {
      if (Array.isArray(this.player.trackArtists)) {
        return this.player.trackArtists.join(', ')
      }
      return this.player.trackArtists || ''
    }
  },

  methods: {
    getNowPlayingClass() {
      return this.player.playing ? 'now-playing--active' : 'now-playing--idle'
    },
    async updateColors (imageUrl) {
      if (!imageUrl) return

      // If the artwork comes from the Sonos speaker (:1400) it lacks CORS headers.
      // We fetch it ourselves, turn it into a blob URL (same-origin) and hand that to Vibrant.
      let sourceForVibrant = imageUrl

      if (imageUrl.includes(':1400/')) {
        try {
          const res      = await fetch(imageUrl)
          const blob     = await res.blob()
          sourceForVibrant = URL.createObjectURL(blob)
        } catch (e) {
          console.error('Could not fetch local Sonos artwork:', e)
        }
      }

      try {
        const palette = await Vibrant.from(sourceForVibrant)
                                    .quality(1)
                                    .clearFilters()
                                    .getPalette()

        const swatches = Object.values(palette).filter(Boolean)
        if (!swatches.length) return

        const swatch     = swatches[Math.floor(Math.random() * swatches.length)]
        const bgColor    = swatch.getHex()
        const textColor  = this.calculateTextColor(bgColor)

        document.documentElement.style.setProperty('--color-text-primary', textColor)
        document.documentElement.style.setProperty('--colour-background-now-playing', bgColor)
      } catch (err) {
        console.error('Vibrant failed:', err)
      } finally {
        // Clean up any blob URL we created
        if (sourceForVibrant.startsWith('blob:')) {
          URL.revokeObjectURL(sourceForVibrant)
        }
      }
    },
    calculateTextColor(bgHex) {
      const c = bgHex.replace('#', '')
      const rgb = parseInt(c, 16)
      const r = (rgb >> 16) & 0xff
      const g = (rgb >> 8) & 0xff
      const b = (rgb >> 0) & 0xff
      const brightness = 0.299 * r + 0.587 * g + 0.114 * b
      return brightness > 150 ? '#000' : '#fff'
    }
  },

  watch: {
    'player.trackAlbum.image': {
      immediate: true,
      handler(newVal) {
        if (newVal) this.updateColors(newVal)
      }
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>