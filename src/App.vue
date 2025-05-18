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
        trackAlbum: {
          image: '',
          paletteSrc: ''
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

          // Find a zone that is currently playing
          const activeZone = zones.find(zone =>
            zone.members.some(m => m.state.playbackState === 'PLAYING')
          )

          if (activeZone) {
            const coordinator = activeZone.members.find(m => m.coordinator)
            const state = coordinator.state.currentTrack

            /* ── Figure out the speaker’s IP (for Sonos-hosted artwork) ── */
            let sonosIP = coordinator.ip || 'localhost'
            if (!coordinator.ip &&
                coordinator.state.nextTrack &&
                coordinator.state.nextTrack.absoluteAlbumArtUri) {
              try {
                const u = new URL(coordinator.state.nextTrack.absoluteAlbumArtUri)
                sonosIP = u.hostname
              } catch (err) {
                console.error('Sonos IP parse error:', err)
              }
            }

            /* ── Build the artwork URL used in the <img> element ── */
            let image = ''

            // 1. Prefer absoluteAlbumArtUri if it looks like a real URL
            if (state.absoluteAlbumArtUri &&
                state.absoluteAlbumArtUri.startsWith('http')) {
              image = state.absoluteAlbumArtUri

            // 2. Otherwise fall back to albumArtUri
            } else if (state.albumArtUri) {
              image = state.albumArtUri.startsWith('http')
                ? state.albumArtUri
                : `http://${sonosIP}:1400${state.albumArtUri}`
            }

            /* ── NEW: a CORS-friendly copy just for node-vibrant ── */
            const paletteSrc =
              image.includes(':1400/') || image.includes(`://${sonosIP}:1400`)
                ? `http://localhost:5005/album-art?url=${encodeURIComponent(image)}`
                : image

            /* ── Update the reactive player object ── */
            this.player = {
              playing: true,
              trackTitle: state.title,
              trackArtists: [state.artist],
              trackAlbum: { image, paletteSrc }
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