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
    /* ─────────────────────────────────────────────────────────────
       Poll Sonos every 2 s, but tolerate short TRANSITIONING gaps
       before we say “nothing is playing”.
       ──────────────────────────────────────────────────────────── */
    async pollSonosState() {
      const GRACE_MS = 5000           // 5-second cushion
      let   lastActive = 0
      let   lastPlayer = this.player

      const checkState = async () => {
        try {
          const res   = await fetch('http://localhost:5005/zones')
          const zones = await res.json()

          /* 1 ─ Find a zone whose playbackState is PLAYING or TRANSITIONING */
          const activeZone = zones.find(zone =>
            zone.members.some(m =>
              ['PLAYING', 'TRANSITIONING'].includes(m.state.playbackState)
            )
          )

          if (activeZone) {
            /* 2 ─ Build the player object exactly as before */
            const coordinator = activeZone.members.find(m => m.coordinator)
            const trackState  = coordinator.state.currentTrack

            /* ── Discover the speaker’s IP ─────────────────────────────── */
            let sonosIP = coordinator.ip || ''

            // ②  pick any other member that *does* have an IP
            if (!sonosIP) {
              const m = activeZone.members.find(m => m.ip)
              if (m) sonosIP = m.ip
            }

            // ③  fall back to host name in absoluteAlbumArtUri (if present)
            if (!sonosIP && coordinator.state.nextTrack?.absoluteAlbumArtUri) {
              try {
                sonosIP = new URL(
                  coordinator.state.nextTrack.absoluteAlbumArtUri
                ).hostname
              } catch (_) { /* ignore parse errors */ }
            }

            /* ── Build the <img> artwork URL ───────────────────────────── */
            let image = ''

            const hasProg = trackState.albumArtUri
              ? trackState.albumArtUri.includes('x-sonosprog-spotify')
              : false

            if (
              (hasProg || !trackState.albumArtUri) &&
              trackState.absoluteAlbumArtUri?.startsWith('http')
            ) {
              // artist-radio or albumArtUri missing → use absoluteAlbumArtUri
              image = trackState.absoluteAlbumArtUri
            } else if (trackState.albumArtUri) {
              if (trackState.albumArtUri.startsWith('http')) {
                // already a full URL
                image = trackState.albumArtUri
              } else if (sonosIP) {
                // relative path hosted on the speaker
                image = `http://${sonosIP}:1400${trackState.albumArtUri}`
              } else if (trackState.absoluteAlbumArtUri?.startsWith('http')) {
                // last-resort fallback – still shows artwork
                image = trackState.absoluteAlbumArtUri
              }
            }

            /* CORS-safe copy for node-vibrant */
            const paletteSrc =
              image.includes(':1400/') || image.includes(`://${sonosIP}:1400`)
                ? `http://localhost:5005/album-art?url=${encodeURIComponent(image)}`
                : image

            /* Update reactive data */
            this.player = {
              playing: true,
              trackTitle: trackState.title,
              trackArtists: [trackState.artist],
              trackAlbum: { image, paletteSrc }
            }

            /* remember active time & snapshot */
            lastActive = Date.now()
            lastPlayer = this.player
          } else {
            /* 3 ─ No active zone; use grace period to avoid flicker */
            if (Date.now() - lastActive > GRACE_MS) {
              this.player.playing = false      // show idle
            } else {
              this.player = lastPlayer         // keep showing last track
            }
          }
        } catch (err) {
          console.error('Sonos API error:', err)
        }
      }

      checkState()
      setInterval(checkState, 2000)
    }
  }
}
</script>
