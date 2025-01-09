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
          @load="onImageLoad"
        />
      </div>
      <div class="now-playing__details">
        <!-- Give these elements refs for resizing logic -->
        <h1 ref="trackElement" class="now-playing__track">
          {{ player.trackTitle }}
        </h1>
        <h2 ref="artistElement" class="now-playing__artists">
          {{ getTrackArtists }}
        </h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant';
import props from '@/utils/props.js';

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    };
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ');
    }
  },

  mounted() {
    console.log('NowPlaying component mounted');
    this.setDataInterval();
    // Attempt text sizing once the component is mounted
    this.$nextTick(() => {
      console.log('Initial $nextTick in mounted');
      this.resizeAllText();
    });
  },

  beforeDestroy() {
    console.log('NowPlaying component is being destroyed');
    clearInterval(this.pollPlaying);
  },

  methods: {
    onImageLoad() {
      console.log('Album artwork loaded');
    },
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      console.log('Fetching now playing data from Spotify...');
      let data = {};

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        );

        if (!response.ok) {
          console.error(`Error fetching now playing data: ${response.status}`);
          throw new Error(`An error has occured: ${response.status}`);
        }

        if (response.status === 204) {
          console.log('No content returned from Spotify (204)');
          data = this.getEmptyPlayer();
          this.playerData = data;
          this.$nextTick(() => {
            console.log('Emitting spotifyTrackUpdated (204)');
            this.$emit('spotifyTrackUpdated', data);
          });
          return;
        }

        data = await response.json();
        console.log('Successfully fetched now playing data:', data);
        this.playerResponse = data;
      } catch (error) {
        console.error('Error fetching now playing data:', error);
        this.handleExpiredToken();
        data = this.getEmptyPlayer();
        this.playerData = data;
        this.$nextTick(() => {
          console.log('Emitting spotifyTrackUpdated due to error');
          this.$emit('spotifyTrackUpdated', data);
        });
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle';
      return `now-playing--${playerClass}`;
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        console.log('No album image to get colours from.');
        return;
      }
      console.log('Fetching album colours...');
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          console.log('Album palette received:', palette);
          this.handleAlbumPalette(palette);
        })
        .catch(error => {
          console.error('Error getting album colours:', error);
        });
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      };
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying);
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying();
      }, 2500);
      console.log('Set data polling interval.');
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      console.log('Setting app colours:', this.colourPalette);
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      );
      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      );
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      console.log('Handling new Spotify track data:', this.playerResponse);
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        console.warn('Received an error from Spotify, likely an expired token.');
        this.handleExpiredToken();
        return;
      }

      if (this.playerResponse.is_playing === false) {
        console.log('Spotify is not playing.');
        this.playerData = this.getEmptyPlayer();
        return;
      }

      if (this.playerResponse.item?.id === this.playerData.trackId) {
        console.log('Track ID has not changed, skipping update.');
        return;
      }

      const newPlayerData = {
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
      };
      console.log('New player data:', newPlayerData);
      this.playerData = newPlayerData;
    },

    /**
     * Handle newly stored colour palette.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item;
        })
        .map(colour => {
          const bgHex = palette[colour].getHex();
          let textColor = palette[colour].getTitleTextColor();

          const brightness = this.calculateBrightness(bgHex);
          const threshold = 150;
          if (brightness > threshold) {
            textColor = '#000';
          }

          return {
            text: textColor,
            background: bgHex
          };
        });

      this.swatches = albumColours;
      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)];

      this.$nextTick(() => {
        console.log('Setting app colours in handleAlbumPalette $nextTick');
        this.setAppColours();
      });
    },

    // Helper function to compute brightness for color palette usage
    calculateBrightness(hex) {
      const c = hex.replace('#', '');
      const rgb = parseInt(c, 16);
      const r = (rgb >> 16) & 0xff;
      const g = (rgb >> 8) & 0xff;
      const b = (rgb >> 0) & 0xff;
      return 0.299 * r + 0.587 * g + 0.114 * b;
    },

    /**
     * Attempt to resize track and artist text to fit their containers.
     */
    resizeAllText() {
      console.log('Attempting to resize all text');
      this.resizeTextToFit(this.$refs.trackElement, 84, 16);
      this.resizeTextToFit(this.$refs.artistElement, 80, 16);
    },

    /**
     * Dynamically resize text until it fits the container’s dimensions without scrolling.
     * @param {HTMLElement} el        The element whose text needs resizing
     * @param {number} initialSize    The starting font size (e.g. 84)
     * @param {number} minSize        The smallest allowed font size (e.g. 16)
     */
    resizeTextToFit(el, initialSize, minSize) {
      if (!el) {
        console.warn('Element for resizing is null:', el);
        return;
      }

      const container = el.parentElement;
      if (!container) {
        console.warn('Parent container for resizing element is null:', el);
        return;
      }

      let currentSize = initialSize;
      el.style.fontSize = currentSize + 'px';

      console.log(`Resizing element: ${el.className}, initialSize: ${initialSize}`);
      while (
        (el.scrollHeight > container.clientHeight || el.scrollWidth > container.clientWidth) &&
        currentSize > minSize
      ) {
        currentSize--;
        el.style.fontSize = currentSize + 'px';
        console.log(`- Reduced font size of ${el.className} to ${currentSize}px`);
      }
      console.log(`Finished resizing ${el.className}, final size: ${currentSize}px`);
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying);
      this.$emit('requestRefreshToken');
      console.warn('Access token expired, requesting refresh.');
    }
  },

  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      console.log('Auth object updated:', newVal);
      if (newVal.status === false) {
        clearInterval(this.pollPlaying);
        console.log('Auth status is false, clearing polling interval.');
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function(newVal, oldVal) {
      console.log('playerResponse updated:', newVal);
      this.handleNowPlaying();
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function(newVal, oldVal) {
      console.log('playerData updated:', newVal);
      this.$emit('spotifyTrackUpdated', newVal);

      // After updating the track info, try resizing the text and extracting album colors
      this.$nextTick(() => {
        console.log('Running resizeAllText and getAlbumColours in playerData $nextTick');
        this.resizeAllText();
        this.getAlbumColours();
      });
    }
  }
};
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>