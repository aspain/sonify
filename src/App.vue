<template>
  <div id="app">
    <NowPlaying :player="player" />
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
        trackAlbum: {
          image: ''
        }
      }
    }
  },
  mounted() {
    this.pollSonosState()
  },
  methods: {
    async pollSonosState() {
      const checkState = async () => {
        try {
          const response = await fetch('http://localhost:5005/zones')
          const zones = await response.json()
          
          const activeZone = zones.find(zone => 
            zone.members.some(member => member.state.playbackState === 'PLAYING')
          )

          if (activeZone) {
            const coordinator = activeZone.members.find(m => m.coordinator)
            const state = coordinator.state.currentTrack

            // If albumArtUri is a relative URL (from /getaa), use it; otherwise, use the provided full URL.
            let imageUrl = ''
            if (state.albumArtUri && state.albumArtUri.startsWith('/getaa')) {
              imageUrl = `http://localhost:5005${state.albumArtUri}`
            } else {
              imageUrl = state.albumArtUri || state.absoluteAlbumArtUri || ''
            }

            this.player = {
              playing: true,
              trackTitle: state.title,
              trackArtists: [state.artist],
              trackAlbum: {
                image: imageUrl
              }
            }
          } else {
            this.player.playing = false
          }
        } catch (error) {
          console.error('Sonos API error:', error)
        }
      }

      // Initial check and then every 2 seconds
      checkState()
      setInterval(checkState, 2000)
    }
  }
}
</script>
