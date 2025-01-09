<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <!-- The album image triggers color extraction & text resizing once loaded -->
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
          @load="handleAlbumImageLoad"
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
      swatches: []
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
    // Start polling Spotify
    this.setDataInterval()

    // Attempt an initial text sizing once mounted
    this.$nextTick(() => {
      this.resizeAllText()
    })
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Fetch the current Spotify track data.
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

        // Spotify returns 204 if nothing is playing
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
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Set the polling interval to update now-playing data regularly.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Return an appropriate class for the now-playing container.
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Create a default empty object for "nothing playing."
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
     * Handle the new Spotify track info contained in playerResponse.
     */
    handleNowPlaying() {
      // Check if token is invalid
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

      // If track ID hasn't changed, do nothing
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      // Otherwise update with new track data
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map((artist) => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Once album image is loaded, extract palette & resize text.
     */
    handleAlbumImageLoad() {
      this.getAlbumColours()
      this.$nextTick(() => {
        this.resizeAllText()
      })
    },

    /**
     * Use node-vibrant to extract colors from album cover.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return
      }
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then((palette) => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Store extracted color palette and update CSS variables.
     */
    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter((item) => item) // remove any null or undefined keys
        .map((colour) => {
          const bgHex = palette[colour].getHex()
          let textColor = palette[colour].getTitleTextColor()

          // Adjust if background is bright
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

    /**
     * Update document-level CSS variables.
     */
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
     * Calculate brightness from a hex color.
     */
    calculateBrightness(hex) {
      const c = hex.replace('#', '')
      const rgb = parseInt(c, 16)
      const r = (rgb >> 16) & 0xff
      const g = (rgb >> 8) & 0xff
      const b = (rgb >> 0) & 0xff
      return 0.299 * r + 0.587 * g + 0.114 * b
    },

    /**
     * Resize both track and artist text to fit their containers.
     */
    resizeAllText() {
      this.resizeTextToFit(this.$refs.trackElement, 84, 16)
      this.resizeTextToFit(this.$refs.artistElement, 80, 16)
    },

    /**
     * Repeatedly shrink text until it fits the container.
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
    },

    /**
     * Handle a Spotify token that is expired or invalid.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    /**
     * When Spotify auth fails, stop polling.
     */
    auth(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Whenever the Spotify response changes, we update the local data.
     */
    playerResponse() {
      this.handleNowPlaying()
    },

    /**
     * Whenever the track title changes, re-run text resizing.
     */
    'playerData.trackTitle'() {
      this.$nextTick(() => {
        this.resizeAllText()
      })
    },

    /**
     * Whenever the artist list changes, re-run text resizing.
     */
    'playerData.trackArtists'() {
      this.$nextTick(() => {
        this.resizeAllText()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
