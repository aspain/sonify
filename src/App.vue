<template>
  <div id="app">
    <NowPlaying :player="player" />
  </div>
</template>

<script>
import NowPlaying from '@/components/NowPlaying'

export default {
  name: 'App',
  components: { NowPlaying },
  data() {
    return {
      player: {
        playing: false,
        trackArtists: [],
        trackTitle: '',
        trackAlbum: { image: '' }
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

          // Find any zone with a playing member
          const activeZone = zones.find(zone =>
            zone.members.some(member => member.state.playbackState === 'PLAYING')
          )

          if (activeZone) {
            const coordinator = activeZone.members.find(m => m.coordinator)
            const state = coordinator.state.currentTrack

            // Try to get the Sonos IP (either directly or from the next track’s absolute URL)
            let sonosIP = 'localhost'
            if (coordinator.ip) {
              sonosIP = coordinator.ip
            } else if (
              coordinator.state.nextTrack &&
              coordinator.state.nextTrack.absoluteAlbumArtUri
            ) {
              try {
                const url = new URL(coordinator.state.nextTrack.absoluteAlbumArtUri)
                sonosIP = url.hostname
              } catch (err) {
                console.error('Error parsing Sonos IP:', err)
              }
            }

            // Determine the correct album art URL:
            // – If albumArtUri is a full URL (Spotify Connect), use it directly.
            // – Otherwise (Sonos app), prefix with your Sonos IP and port 1400.
            let image = ''
            if (state.albumArtUri) {
              if (state.albumArtUri.startsWith('http')) {
                image = state.albumArtUri
              } else {
                image = `http://${sonosIP}:1400${state.albumArtUri}`
              }
            } else if (state.absoluteAlbumArtUri) {
              image = state.absoluteAlbumArtUri
            }

            this.player = {
              playing: true,
              trackTitle: state.title,
              trackArtists: [state.artist],
              trackAlbum: { image }
            }
          } else {
            this.player.playing = false
          }
        } catch (error) {
          console.error('Sonos API error:', error)
        }
      }

      checkState()
      setInterval(checkState, 2000)
    }
  }
}
</script>
