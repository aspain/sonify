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

            let imageUrl = ''

            // Use the absolute URL if it doesn’t point to a mosaic image
            if (state.absoluteAlbumArtUri && state.absoluteAlbumArtUri.indexOf('mosaic.scdn.co') === -1) {
              imageUrl = state.absoluteAlbumArtUri
            } else if (state.albumArtUri) {
              // Build the URL if albumArtUri is relative
              imageUrl = state.albumArtUri.startsWith('http')
                ? state.albumArtUri
                : `http://localhost:5005${state.albumArtUri}`
            }

            // If the imageUrl is still the mosaic image, try to fetch proper album art from Spotify
            if (imageUrl.indexOf('mosaic.scdn.co') !== -1 && state.trackUri) {
              try {
                const spotifyResponse = await fetch(
                  `http://localhost:5006/spotifyalbumart?uri=${encodeURIComponent(state.trackUri)}`
                )
                if (spotifyResponse.ok) {
                  const data = await spotifyResponse.json()
                  if (data.albumArtUrl) {
                    imageUrl = data.albumArtUrl
                  }
                }
              } catch (err) {
                console.error('Spotify album art fetch error:', err)
              }
            }

            this.player = {
              playing: true,
              trackTitle: state.title,
              trackArtists: [state.artist],
              trackAlbum: { image: imageUrl }
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
