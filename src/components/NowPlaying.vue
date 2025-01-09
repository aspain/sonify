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
    console.log('[mounted] Component mounted. Setting data interval and resizing text.')
    this.setDataInterval()
    // Guard check: only resize if the ref elements actually exist
    this.$nextTick(() => {
      if (this.$refs.trackElement && this.$refs.artistElement) {
        this.resizeAllText()
      } else {
        console.log('[mounted] trackElement or artistElement not found yet. Skipping resize.')
      }
    })
  },

  beforeDestroy() {
    console.log('[beforeDestroy] Clearing pollPlaying interval.')
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Fetch the current playing track from Spotify.
     */
    async getNowPlaying() {
      console.log('[getNowPlaying] Fetching track details...')
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
          console.error(`[getNowPlaying] Response not OK: ${response.status}`)
          throw new Error(`An error has occured: ${response.status}`)
        }

        if (response.status === 204) {
          console.log('[getNowPlaying] No track playing (204). Setting empty player data.')
          data = this.getEmptyPlayer()
          this.playerData = data
          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })
          return
        }

        data = await response.json()
        console.log('[getNowPlaying] Data received:', data)
        this.playerResponse = data
      } catch (error) {
        console.error('[getNowPlaying] Error fetching track details:', error)
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Determine the class for now-playing wrapper based on state.
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Extract album colors using node-vibrant.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        console.log('[getAlbumColours] No album image found. Skipping color extraction.')
        return
      }
      console.log('[getAlbumColours] Extracting album colors from image:', this.player.trackAlbum.image)
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
        .catch(err => {
          console.error('[getAlbumColours] Error extracting palette:', err)
        })
    },

    /**
     * Return empty object for idle player.
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
     * Poll Spotify for updates every 2.5s.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      console.log('[setDataInterval] Starting poll at 2.5s intervals.')
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Apply new colors to the app after we get them from Vibrant.
     */
    setAppColours() {
      console.log('[setAppColours] Setting text and background colors:', this.colourPalette)
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
     * Update local playerData based on new Spotify data.
     */
    handleNowPlaying() {
      console.log('[handleNowPlaying] Handling new playerResponse.')
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        console.warn('[handleNowPlaying] Token expired or invalid. Handling expired token.')
        this.handleExpiredToken()
        return
      }

      if (this.playerResponse.is_playing === false) {
        console.log('[handleNowPlaying] No track is playing. Setting empty player.')
        this.playerData = this.getEmptyPlayer()
        return
      }

      if (this.playerResponse.item?.id === this.playerData.trackId) {
        console.log('[handleNowPlaying] Same track ID as before. No update needed.')
        return
      }

      console.log('[handleNowPlaying] Detected a new track. Updating local playerData.')
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
     * Process Vibrant palette.
     */
    handleAlbumPalette(palette) {
      console.log('[handleAlbumPalette] Palette received:', palette)
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
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
      console.log('[handleAlbumPalette] Randomly chosen palette color:', this.colourPalette)

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * Compute brightness for color picking.
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
     * Resize both track and artist text to fit.
     */
    resizeAllText() {
      console.log('[resizeAllText] Resizing track and artist text.')
      // Only call resizeTextToFit if refs are available:
      if (this.$refs.trackElement) {
        this.resizeTextToFit(this.$refs.trackElement, 84, 16)
      } else {
        console.log('[resizeAllText] trackElement ref not found.')
      }

      if (this.$refs.artistElement) {
        this.resizeTextToFit(this.$refs.artistElement, 80, 16)
      } else {
        console.log('[resizeAllText] artistElement ref not found.')
      }
    },

    /**
     * Dynamically resize text to fit its container.
     */
    resizeTextToFit(el, initialSize, minSize) {
      if (!el) {
        console.warn('[resizeTextToFit] No element provided.')
        return
      }

      const label = el === this.$refs.trackElement ? 'Track' : 'Artist'
      const container = el.parentElement
      let currentSize = initialSize
      el.style.fontSize = currentSize + 'px'
      console.log(`[resizeTextToFit] Starting ${label} font size at ${currentSize}px.`)

      while (
        (el.scrollHeight > container.clientHeight || el.scrollWidth > container.clientWidth) &&
        currentSize > minSize
      ) {
        currentSize--
        el.style.fontSize = currentSize + 'px'
      }

      console.log(`[resizeTextToFit] Final ${label} font size is ${currentSize}px (minSize was ${minSize}).`)
    },

    /**
     * Handle expired or invalid Spotify token.
     */
    handleExpiredToken() {
      console.log('[handleExpiredToken] Clearing interval and requesting token refresh.')
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    /**
     * Watch auth changes.
     */
    auth: function(oldVal, newVal) {
      console.log('[watch:auth] Auth changed:', newVal)
      if (newVal.status === false) {
        console.log('[watch:auth] Auth status false. Clearing pollPlaying.')
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch for new data from Spotify (raw playerResponse).
     */
    playerResponse: function() {
      console.log('[watch:playerResponse] playerResponse changed. Calling handleNowPlaying.')
      this.handleNowPlaying()
    },

    /**
     * Watch local playerData for changes.
     */
    playerData: function() {
      console.log('[watch:playerData] playerData changed:', this.playerData)
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        console.log('[watch:playerData] Attempting text resize and color extraction.')
        this.resizeAllText()
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
