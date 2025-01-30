<template>
  <div id="app">
    <div v-if="player.playing" class="now-playing now-playing--active">
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track">{{ player.trackTitle }}</h1>
        <h2 class="now-playing__artists">{{ getTrackArtists }}</h2>
      </div>
    </div>
    <div v-else class="now-playing now-playing--idle">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

export default {
  name: 'NowPlaying',

  data() {
    return {
      pollIntervalId: null,
      playerData: this.getEmptyPlayer()
    }
  },

  computed: {
    getTrackArtists() {
      return this.playerData.trackArtists.join(', ')
    },
    player() {
      // This just returns the local data as "player"
      return this.playerData
    }
  },

  mounted() {
    this.fetchSonosState() // initial fetch
    this.pollIntervalId = setInterval(this.fetchSonosState, 2500) // poll every 2.5s
  },

  beforeDestroy() {
    clearInterval(this.pollIntervalId)
  },

  methods: {
    getEmptyPlayer() {
      return {
        playing: false,
        trackTitle: '',
        trackArtists: [],
        trackAlbum: { image: '' }
      }
    },

    async fetchSonosState() {
      try {
        // 1) Could also fetch /zones and find the first active room,
        //    but let's assume you only care about "Living Room."
        const response = await fetch('http://192.168.4.96:5005/Living Room/state')
        const data = await response.json()

        // 2) Check if "playbackState" === "PLAYING"
        if (data.playbackState === 'PLAYING') {
          this.playerData.playing = true
          this.playerData.trackTitle = data.currentTrack.title || ''
          this.playerData.trackArtists = [data.currentTrack.artist || '']
          // 3) Build albumArt URL
          this.playerData.trackAlbum.image =
            `http://192.168.4.96:5005${data.currentTrack.albumArtUri}`
        } else {
          // Not playing
          this.playerData = this.getEmptyPlayer()
        }

        // 4) Use Vibrant to set background color
        if (this.playerData.playing && this.playerData.trackAlbum.image) {
          this.setAlbumColors(this.playerData.trackAlbum.image)
        }
      } catch (err) {
        console.error(err)
        this.playerData = this.getEmptyPlayer()
      }
    },

    async setAlbumColors(imageUrl) {
      try {
        const palette = await Vibrant.from(imageUrl).getPalette()
        // pick a random swatch
        const swatchArray = Object.keys(palette).map(key => palette[key])
        const chosenSwatch =
          swatchArray[Math.floor(Math.random() * swatchArray.length)]
        const bgHex = chosenSwatch.getHex()
        const textColor = chosenSwatch.getTitleTextColor()

        document.documentElement.style.setProperty('--color-text-primary', textColor)
        document.documentElement.style.setProperty('--colour-background-now-playing', bgHex)
      } catch (error) {
        console.error('Vibrant error:', error)
      }
    }
  }
}
</script>

<style scoped>
/* Keep or modify your existing styles. */
</style>
