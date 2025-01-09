<template>
  <div id="app">
    <!-- Use playerData here rather than player -->
    <div
      v-if="playerData.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="playerData.trackAlbum.image"
          :alt="playerData.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 ref="trackElement" class="now-playing__track">
          {{ playerData.trackTitle }}
        </h1>
        <h2 ref="artistElement" class="now-playing__artists">
          {{ getPlayerDataArtists }}
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
    // keep this around for other references,
    // but we won't directly render from it
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      // This is the object you actually mutate in watchers:
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * If you still want a computed for the prop,
     * you can keep it, but we’ll rely on playerData in the template.
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },

    /**
     * A simple computed that returns a comma-separated
     * list of artists from our local playerData object.
     */
    getPlayerDataArtists() {
      return this.playerData.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
    // Attempt text sizing once the component is mounted
    this.$nextTick(() => {
      this.resizeAllText()
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

    getNowPlayingClass() {
      // Compare against our local playerData for class
      const playerClass = this.playerData.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    getAlbumColours() {
      if (!this.playerData.trackAlbum?.image) return
      Vibrant.from(this.playerData.trackAlbum.image)
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

      // Only update local playerData if track has changed
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
      let albumColours = Object.keys(palette)
        .filter(item => item)
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

    resizeAllText() {
      this.resizeTextToFit(this.$refs.trackElement, 84, 16)
      this.resizeTextToFit(this.$refs.artistElement, 80, 16)
    },

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

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our local playerData object
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      this.$nextTick(() => {
        this.resizeAllText()
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
