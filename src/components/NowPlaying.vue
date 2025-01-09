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
        <h1 class="now-playing__track" ref="trackTitle">{{ player.trackTitle }}</h1>
        <h2 class="now-playing__artists" ref="trackArtists">{{ getTrackArtists }}</h2>
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
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
    // Just a one-time check at mount (if there's data)
    this.$nextTick(() => {
      this.adjustAllTextSizes()
    })
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
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

        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
            this.adjustAllTextSizes()
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
          this.adjustAllTextSizes()
        })
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

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

    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
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

    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()
        return
      }

      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()
        return
      }

      // If track hasn't changed, no need to re-update
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter(item => item != null)
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

    calculateBrightness(hex) {
      const c = hex.replace('#', '')
      const rgb = parseInt(c, 16)
      const r = (rgb >> 16) & 0xff
      const g = (rgb >> 8) & 0xff
      const b = (rgb >> 0) & 0xff
      return 0.299 * r + 0.587 * g + 0.114 * b
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    },

    /** 
     * Adjust text sizes for trackTitle and trackArtists 
     * so they don't overflow their parent container.
     */
    adjustAllTextSizes() {
      this.$nextTick(() => {
        this.fitTextToContainer(this.$refs.trackTitle, 24, 84)
        this.fitTextToContainer(this.$refs.trackArtists, 24, 80)
      })
    },

    /**
     * A binary-search approach to find the largest font size 
     * that doesn't overflow the element's parent container.
     */
    fitTextToContainer(el, minFont, maxFont) {
      if (!el || !el.parentElement) return

      // 1) Clear any inline font-size from a previous track
      el.style.fontSize = ''

      // 2) Save the parent’s size for reference
      const parentRect = el.parentElement.getBoundingClientRect()

      // 3) A helper to check if the text overflows horizontally or vertically
      const isOverflowing = fontPx => {
        el.style.fontSize = fontPx + 'px'
        const rect = el.getBoundingClientRect()
        return (rect.width > parentRect.width || rect.height > parentRect.height)
      }

      // 4) Binary search from minFont..maxFont
      let low = minFont
      let high = maxFont
      let bestFit = low

      while (low <= high) {
        const mid = Math.floor((low + high) / 2)
        if (isOverflowing(mid)) {
          // Too big
          high = mid - 1
        } else {
          // Fits, record it, try bigger
          bestFit = mid
          low = mid + 1
        }
      }

      // 5) Apply the best fit
      el.style.fontSize = bestFit + 'px'
    }
  },

  watch: {
    auth(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },
    playerResponse() {
      this.handleNowPlaying()
    },
    /**
     * Each time we get a new track, 
     * re-check the text sizes so each new track can start big 
     * and shrink if necessary.
     */
    playerData() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      this.$nextTick(() => {
        this.getAlbumColours()
        this.adjustAllTextSizes()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
