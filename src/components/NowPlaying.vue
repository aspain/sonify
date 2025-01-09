<template>
  <div :key="renderKey" id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          ref="coverImage"
          @load="onCoverImageLoaded"
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 ref="trackElement" class="now-playing__track">
          {{ player.trackTitle }}
        </h1>
        <h2 ref="artistElement" class="now-playing__artists">
          {{ getTrackArtists }}
        </h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: null,
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      // A key to force re-render whenever the track changes significantly
      renderKey: 0
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
    // Attempt text sizing once mounted
    this.$nextTick(() => {
      this.resizeAllText()
    })
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Fired whenever the album cover finishes loading.
     * Ensures container dimensions are updated before resizing text.
     */
    onCoverImageLoaded() {
      // A short delay ensures the browser has updated layout before measuring
      requestAnimationFrame(() => {
        this.resizeAllText()
      })
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Make the network request to Spotify for the current track.
     */
    async getNowPlaying() {
      let data = {}
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        // No content means nothing is playing
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data
          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })
          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        // Likely an expired token
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Return the appropriate class for now-playing state.
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Generate a default, “nothing playing” object.
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    /**
     * Handle token expiration.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    },

    /**
     * Process newly-fetched data from Spotify.
     */
    handleNowPlaying() {
      // If Spotify says our token is invalid
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()
        return
      }

      // If nothing is playing
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()
        return
      }

      // If it's the same track, do nothing
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      // Otherwise, store the new track
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Extract color palette from album cover.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) return

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter(Boolean)
        .map(key => {
          const bgHex = palette[key].getHex()
          let textColor = palette[key].getTitleTextColor()

          const brightness = this.calculateBrightness(bgHex)
          const threshold = 150
          if (brightness > threshold) {
            textColor = '#000'
          }

          return {
            text: textColor,
            background: bgHex
          }
        })

      this.swatches = albumColours
      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )
      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Calculates brightness so we can pick black/white text automatically.
     */
    calculateBrightness(hex) {
      const c = hex.replace('#', '')
      const rgb = parseInt(c, 16)
      const r = (rgb >> 16) & 0xff
      const g = (rgb >> 8) & 0xff
      const b = rgb & 0xff
      return 0.299 * r + 0.587 * g + 0.114 * b
    },

    /**
     * Resize both track & artist text to fit their containers.
     */
    resizeAllText() {
      // `Log`s for debugging
      console.log('Resizing text...')
      this.resizeTextToFit(this.$refs.trackElement, 84, 16)
      this.resizeTextToFit(this.$refs.artistElement, 80, 16)
    },

    /**
     * Resize until it fits the container without overflowing.
     */
    resizeTextToFit(el, initialSize, minSize) {
      if (!el) return
      const container = el.parentElement
      let currentSize = initialSize
      el.style.fontSize = currentSize + 'px'

      while (
        (el.scrollHeight > container.clientHeight ||
          el.scrollWidth > container.clientWidth) &&
        currentSize > minSize
      ) {
        currentSize--
        el.style.fontSize = currentSize + 'px'
      }
    }
  },

  watch: {
    /**
     * Whenever Spotify’s response object changes
     */
    playerResponse() {
      this.handleNowPlaying()
    },

    /**
     * Our local track data changes whenever handleNowPlaying updates it
     */
    playerData: {
      immediate: true,
      handler(newVal, oldVal) {
        // Let parent know
        this.$emit('spotifyTrackUpdated', newVal)

        // If track changed, force re-render
        if (newVal.trackId && oldVal && newVal.trackId !== oldVal.trackId) {
          this.renderKey++
        }

        // Wait a moment for DOM updates, then resize text & get palette
        this.$nextTick(() => {
          setTimeout(() => {
            this.resizeAllText()
            this.getAlbumColours()
          }, 50)
        })
      }
    },

    /**
     * If our token becomes invalid, stop polling
     */
    auth(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    }
  }
}
</script>

<!-- No 'scoped' to ensure accurate container sizing -->
<style src="@/styles/components/now-playing.scss" lang="scss"></style>
