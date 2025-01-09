<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
          @load="handleAlbumImageLoad" <!-- Image load event -->
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
      pollPlaying: '',
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
    this.setDataInterval()
    // Attempt initial text sizing once mounted
    this.$nextTick(() => {
      this.resizeAllText()
    })
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
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

        // If nothing is playing
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
     * Get the Now Playing element class.
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return
      }
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Return a formatted empty object for an idle player.
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
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Apply extracted album colors to the document.
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
     * Handle newly updated Spotify Tracks in playerResponse.
     */
    handleNowPlaying() {
      // Check for invalid token
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

      // If the track ID hasn't changed, do nothing
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      // Otherwise update to the new track
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
     * Handle newly stored colour palette.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => item) // filter out null
        .map(colour => {
          const bgHex = palette[colour].getHex()
          let textColor = palette[colour].getTitleTextColor()
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

    // Helper to compute brightness for color usage
    calculateBrightness(hex) {
      const c = hex.replace('#', '')
      const rgb = parseInt(c, 16)
      const r = (rgb >> 16) & 0xff
      const g = (rgb >> 8) & 0xff
      const b = (rgb >> 0) & 0xff
      return 0.299 * r + 0.587 * g + 0.114 * b
    },

    /**
     * Dynamically resize both text elements to fit their containers.
     */
    resizeAllText() {
      this.resizeTextToFit(this.$refs.trackElement, 84, 16)
      this.resizeTextToFit(this.$refs.artistElement, 80, 16)
    },

    /**
     * Resize text until it fits without causing scrollbars.
     * @param {HTMLElement} el
     * @param {number} initialSize
     * @param {number} minSize
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
     * Handle image load: do color extraction and text resizing.
     */
    handleAlbumImageLoad() {
      // Extract album colors now that the image is loaded
      this.getAlbumColours()
      // Then resize text
      this.$nextTick(() => {
        this.resizeAllText()
      })
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    /**
     * If Spotify auth fails, stop polling.
     */
    auth: function(newVal, oldVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Whenever the Spotify response changes, update local player data.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Whenever track title changes, resize text.
     */
    'playerData.trackTitle': function() {
      this.$nextTick(() => {
        this.resizeAllText()
      })
    },

    /**
     * Whenever artists array changes, resize text.
     */
    'playerData.trackArtists': function() {
      this.$nextTick(() => {
        this.resizeAllText()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
