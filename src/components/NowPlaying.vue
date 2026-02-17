<template>
  <div id="app">
    <div v-if="player.playing" class="now-playing" :class="getNowPlayingClass()">
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
          crossorigin="anonymous"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>

    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music playing</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

function clamp(n, min = 0, max = 255) {
  return Math.min(max, Math.max(min, n))
}

function hexToRgb(hex) {
  const h = hex.replace('#', '').trim()
  const full = h.length === 3 ? h.split('').map(c => c + c).join('') : h
  const num = parseInt(full, 16)
  return {
    r: (num >> 16) & 255,
    g: (num >> 8) & 255,
    b: num & 255
  }
}

function rgbToHex({ r, g, b }) {
  const toHex = v => clamp(v).toString(16).padStart(2, '0')
  return `#${toHex(r)}${toHex(g)}${toHex(b)}`
}

function lighten(hex, amt = 0.12) {
  const { r, g, b } = hexToRgb(hex)
  return rgbToHex({
    r: Math.round(r + (255 - r) * amt),
    g: Math.round(g + (255 - g) * amt),
    b: Math.round(b + (255 - b) * amt)
  })
}

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
    }
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
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
          this.$emit('spotifyTrackUpdated', data)
          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$emit('spotifyTrackUpdated', data)
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    getAlbumColours() {
      if (!this.player.trackAlbum?.image) return

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
        .catch(() => {
          // If palette extraction fails, keep whatever background you already have.
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
  const baseHex =
    (this.colourPalette && this.colourPalette.background) || '#402830'

  const safeBase = /^#[0-9a-fA-F]{6}$/.test(baseHex)
    ? baseHex
    : '#402830'

  // lighten slightly for top
  const lighten = (hex, amt = 0.12) => {
    const num = parseInt(hex.slice(1), 16)
    const r = Math.min(255, Math.round(((num >> 16) & 255) + (255 - ((num >> 16) & 255)) * amt))
    const g = Math.min(255, Math.round(((num >> 8) & 255) + (255 - ((num >> 8) & 255)) * amt))
    const b = Math.min(255, Math.round((num & 255) + (255 - (num & 255)) * amt))
    return `#${((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)}`
  }

  // darken but NOT black
  const darken = (hex, amt = 0.35) => {
    const num = parseInt(hex.slice(1), 16)
    const r = Math.round(((num >> 16) & 255) * (1 - amt))
    const g = Math.round(((num >> 8) & 255) * (1 - amt))
    const b = Math.round((num & 255) * (1 - amt))
    return `#${((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)}`
  }

  const top = lighten(safeBase, 0.12)
  const bottom = darken(safeBase, 0.35) // darker but still color-based

  const gradient = `linear-gradient(
    180deg,
    ${top} 0%,
    ${safeBase} 65%,
    ${bottom} 100%
  )`

  // force text to stay white
  document.documentElement.style.setProperty('--color-text-primary', '#ffffff')
  document.documentElement.style.setProperty(
    '--colour-background-now-playing',
    gradient
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

      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    handleAlbumPalette(palette) {
      const preferred =
        palette.Vibrant ||
        palette.Muted ||
        palette.DarkVibrant ||
        palette.DarkMuted ||
        Object.values(palette).find(s => s)

      if (!preferred) return

      this.colourPalette = {
        text: preferred.getTitleTextColor(),
        background: preferred.getHex()
      }

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    auth(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    playerResponse() {
      this.handleNowPlaying()
    },

    playerData() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
