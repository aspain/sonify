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
    /* ────────────────────────────────────────────────────────────
      Poll Sonos every 2 s, but tolerate short TRANSITIONING gaps
      before we say “nothing is playing”.
      ──────────────────────────────────────────────────────────── */
    async pollSonosState () {
      const GRACE_MS = 5000          // 5-second cushion
      let lastActive = 0             // private variable inside closure
      let lastPlayer = this.player   // remember previous track info

      const checkState = async () => {
        try {
          const res   = await fetch('http://localhost:5005/zones')
          const zones = await res.json()

          /* 1 – find a zone whose state is PLAYING *or* TRANSITIONING */
          const activeZone = zones.find(z =>
            z.members.some(m =>
              ['PLAYING', 'TRANSITIONING'].includes(m.state.playbackState)
            )
          )

          if (activeZone) {
            /* 2 – we’re good; update everything as before */
            const coordinator = activeZone.members.find(m => m.coordinator)
            const state       = coordinator.state.currentTrack

            /* …… (the artwork-resolution & paletteSrc code you already added) …… */

            lastActive = Date.now()
            lastPlayer = this.player      // snapshot the latest
          } else {
            /* 3 – no ACTIVE zone found.  Are we still inside the grace period? */
            if (Date.now() - lastActive > GRACE_MS) {
              this.player.playing = false   // show the idle screen
            } else {
              this.player = lastPlayer      // keep showing last known track
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