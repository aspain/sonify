<template>
  <div id="app">
    <!-- Directly use NowPlaying -->
    <NowPlaying
      :player="player"
      @spotifyTrackUpdated="updateCurrentTrack"
    />
  </div>
</template>

<script>
import NowPlaying from '@/components/NowPlaying'

export default {
  name: 'App',
  components: {
    NowPlaying
  },
  data() {
    return {
      player: {
        playing: false,
        trackArtists: [],
        trackTitle: '',
        trackAlbum: []
      }
    }
  },
  methods: {
    updateCurrentTrack(value) {
      this.player = value
    }
  }
}
</script>

<style>
/* Optional: Keep or remove any global styles if desired */
</style>
