<template>
  <div ref="container" class="video-wrapper filler flexrow-item">
    <video
      ref="player1"
      preload="auto"
      :muted="muted"
      @ended="$emit('play-next')"
    />
    <video
      ref="player2"
      preload="auto"
      :muted="muted"
      @ended="$emit('play-next')"
    />
  </div>
</template>

<script>
/*
 * To play several videos, to avoid blinking effects, it's required to use
 * two video players. When switching from a entity to another, we hide and show
 * players so the blink does not occur.
 * Videos are represented with a list of entities. Each entity has a
 * preview_file_id which is used to build the related movie path. The expected
 * format of entities is:
 * [{
 *    preview_file_id: 'preview-id',
 *    preview_file_extension: 'mp4'
 * }, ...]
 */
import { mapGetters } from 'vuex'
import { floorToFrame, roundToFrame } from '@/lib/video'

export default {
  name: 'raw-video-player',

  components: {},

  props: {
    entities: {
      type: Array,
      default: () => []
    },
    muted: {
      type: Boolean,
      default: false
    },
    isHd: {
      type: Boolean,
      default: false
    },
    isRepeating: {
      type: Boolean,
      default: false
    },
    fullScreen: {
      type: Boolean,
      default: false
    },
    currentPreviewIndex: {
      type: Number,
      default: 0
    },
    name: {
      // Debug purpose
      type: String,
      default: 'main'
    },
    handleIn: {
      type: Number,
      default: 0
    },
    handleOut: {
      type: Number,
      default: 0
    }
  },

  data() {
    return {
      currentPlayer: this.player1,
      isPlaying: false,
      nextPlayer: this.player2,
      playingIndex: 0
    }
  },

  // Video need to be resized after each window size change. It's due
  // to a HTML5 limitation related to video height.
  mounted() {
    this.resetHeight()
    this.player1.addEventListener('loadedmetadata', this.emitLoadedEvent)
    window.addEventListener('resize', this.resetHeight)
    this.$options.currentTimeCalls = []
    if (this.video && this.video.readyState === 4) {
      this.$emit('metadata-loaded', event)
      this.resetHeight()
    }
  },

  beforeDestroy() {
    window.removeEventListener('resize', this.resetHeight)
    this.player1.removeEventListener('loadedmetadata', this.emitLoadedEvent)
  },

  computed: {
    ...mapGetters(['currentProduction']),

    container() {
      return this.$refs.container
    },

    fps() {
      return parseFloat(this.currentProduction.fps || '24')
    },

    player1() {
      return this.$refs.player1
    },

    player2() {
      return this.$refs.player2
    },

    frameDuration() {
      return Math.round((1 / this.fps) * 10000) / 10000
    }
  },

  methods: {
    // Helpers

    emitLoadedEvent(event) {
      this.$emit('metadata-loaded', event)
    },

    getMoviePath(entity) {
      if (entity.preview_file_extension === 'mp4') {
        let previewId
        if (
          this.currentPreviewIndex === 0 ||
          this.currentPreviewIndex > entity.preview_file_previews.length
        ) {
          previewId = entity.preview_file_id
        } else {
          previewId =
            entity.preview_file_previews[this.currentPreviewIndex - 1].id
        }
        if (this.isHd) {
          return `/api/movies/originals/preview-files/${previewId}.mp4`
        } else {
          return `/api/movies/low/preview-files/${previewId}.mp4`
        }
      } else {
        return ''
      }
    },

    getWidth() {
      return this.currentPlayer ? this.currentPlayer.offsetWidth : 0
    },

    getHeight() {
      return this.currentPlayer ? this.currentPlayer.offsetHeight : 0
    },

    getVideoRatio() {
      const height = this.getVideoHeight()
      return height ? this.getVideoWidth() / height : 0
    },

    getVideoWidth() {
      return this.currentPlayer ? this.currentPlayer.videoWidth : 0
    },

    getVideoHeight() {
      return this.currentPlayer ? this.currentPlayer.videoHeight : 0
    },

    clear() {
      if (this.currentPlayer) {
        this.currentPlayer.src = ''
        this.currentPlayer.removeAttribute('src')
        this.currentPlayer.load()
      }
    },

    resetHeight(height) {
      this.$nextTick(() => {
        if (this.currentPlayer) this.currentPlayer.style.height = '0px'
        if (this.nextPlayer) this.nextPlayer.style.height = '0px'
        if (this.container) {
          height = height || this.container.offsetHeight
          if (this.currentPlayer)
            this.currentPlayer.style.height = `${height}px`
          if (this.nextPlayer) this.nextPlayer.style.height = `${height}px`
        }
      })
    },

    // Navigation

    getNextIndex(index) {
      let i = index + 1 >= this.entities.length ? 0 : index + 1
      // While we don't come back to initial entity and we have video previews
      while (
        i !== index &&
        this.entities[i] &&
        this.entities[i].preview_file_extension !== 'mp4'
      ) {
        i++
        if (i >= this.entities.length) i = 0
      }
      return i
    },

    getPreviousIndex(index) {
      let i = index - 1 >= 0 ? index - 1 : this.entities.length - 1
      // While we don't come back to initial entity and we have video previews
      while (
        i !== index &&
        this.entities[i] &&
        this.entities[i].preview_file_extension !== 'mp4'
      ) {
        i--
        if (i < 0) i = this.entities.length
      }
      return i
    },

    goPreviousFrame() {
      if (this.currentPlayer) {
        const time = this.currentTimeRaw
        let newTime = floorToFrame(time - this.frameDuration, this.fps)
        newTime = this._setCurrentTime(newTime)
        const frameNumber = newTime / this.frameDuration
        this.$emit('frame-update', frameNumber)
      }
    },

    goNextFrame() {
      if (this.currentPlayer) {
        const time = this.currentTimeRaw
        let newTime = floorToFrame(time + this.frameDuration, this.fps)
        newTime = this._setCurrentTime(newTime)
        const frameNumber = newTime / this.frameDuration
        this.$emit('frame-update', frameNumber)
      }
    },

    loadPreviousEntity() {
      this.loadEntity(this.getPreviousIndex(this.currentIndex))
      this.$emit('entity-change', this.currentIndex)
    },

    loadNextEntity() {
      const newIndex = this.getNextIndex(this.currentIndex)
      this.loadEntity(newIndex)
      this.$emit('entity-change', this.currentIndex)
    },

    reloadCurrentEntity(silent = false) {
      this.loadEntity(this.currentIndex, this.currentPlayer.currentTime, silent)
    },

    loadEntity(index = 0, currentTime = 0, silent = false) {
      if (index < this.entities.length) {
        const nextIndex = this.getNextIndex(index)
        const entity = this.entities[index]
        const nextEntity = this.entities[nextIndex]

        this.currentIndex = index
        this.currentPlayer = this.player1
        this.nextPlayer = this.player2
        this.currentPlayer.removeEventListener(
          'loadedmetadata',
          this.updateMaxDuration
        )
        this.currentPlayer.addEventListener(
          'loadedmetadata',
          this.updateMaxDuration
        )
        const rate = this.$options.rate || 1

        this.currentPlayer.src = this.getMoviePath(entity)
        this.nextPlayer.src = this.getMoviePath(nextEntity)
        this.currentPlayer.style.display = 'block'
        this.nextPlayer.style.display = 'none'
        this.resetHeight()

        this.setSpeed(rate)
        this._setCurrentTime(currentTime)
        if (!silent) {
          this.$emit('entity-change', this.currentIndex)
        }
      }
    },

    // Playing

    pause() {
      if (this.currentPlayer) {
        this.currentPlayer.pause()
        this.currentPlayer.curentTime = roundToFrame(
          this.currentPlayer.currentTime,
          this.fps
        )
        this.currentTimeRaw = this.currentPlayer.currentTime
        const frameNumber = Math.round(
          this.currentPlayer.currentTime / this.frameDuration
        )
        this.$emit('frame-update', frameNumber)
        clearInterval(this.$options.playLoop)
      }
      this.isPlaying = false
    },

    play() {
      let entity = this.entities[this.currentIndex]
      if (entity) {
        if (!entity.preview_file_id) this.loadNextEntity()
        entity = this.entities[this.currentIndex]
        if (entity.preview_file_id) {
          if (this.currentPlayer) {
            this.runEmitTimeUpdateLoop()
            this.currentPlayer.play()
          }
          this.isPlaying = true
        }
      }
    },

    runEmitTimeUpdateLoop() {
      clearInterval(this.$options.playLoop)
      this.$options.playLoop = setInterval(() => {
        this.updateTime(this.currentPlayer.currentTime)
      }, 1000 / this.fps)
    },

    playNext(handleIn) {
      if (!this.isPlaying) return
      handleIn = handleIn || this.handleIn
      if (this.isRepeating) {
        this.currentPlayer.currentTime = this.handleIn
          ? this.handleIn * this.frameDuration
          : this.frameDuration
        this.currentPlayer.play()
        this.$emit('repeat')
      } else {
        const nextIndex = this.getNextIndex(this.currentIndex)
        this.currentIndex = nextIndex
        this.$emit('entity-change', this.currentIndex)

        if (this.currentPlayer) this.currentPlayer.style.display = 'none'
        if (this.nextPlayer) {
          this.nextPlayer.currentTime = handleIn
            ? handleIn * this.frameDuration
            : this.frameDuration
          this.nextPlayer.style.display = 'block'
          this.nextPlayer.play()
        }

        this.switchPlayers()
        this.updateMaxDuration()
      }
    },

    getCurrentTime() {
      if (this.currentPlayer) {
        return this.currentPlayer.currentTime
      } else {
        return 0
      }
    },

    getLastPushedCurrentTime() {
      const length = this.$options.currentTimeCalls.length
      if (length > 0) {
        return this.$options.currentTimeCalls[length - 1]
      } else {
        return this.getCurrentTime()
      }
    },

    getCurrentTimeRaw(currentTime) {
      return this.currentPlayer.currentTime
    },

    setCurrentTimeRaw(currentTime) {
      if (this.currentPlayer) {
        this.currentPlayer.currentTime = currentTime
      }
    },

    setCurrentFrame(frameNumber) {
      this._setCurrentTime(frameNumber * this.frameDuration)
    },

    _setCurrentTime(newTime) {
      if (!this.$options.currentTimeCalls) {
        this.$options.currentTimeCalls = []
      }
      if (!this.currentPlayer) {
        newTime = 0
      } else if (newTime < 0) {
        newTime = 0
      } else {
        const duration = floorToFrame(this.currentPlayer.duration, this.fps)
        if (newTime > duration) {
          newTime = duration
        }
      }
      this.runSetCurrentTime(newTime)
      return newTime
    },

    runSetCurrentTime(currentTime) {
      if (
        this.currentPlayer &&
        this.currentPlayer.currentTime !== currentTime + this.frameDuration
      ) {
        // tweaks needed because the html video player is messy with frames
        this.currentPlayer.currentTime = currentTime + this.frameDuration + 0.01
        this.onTimeUpdate()
      }
    },

    onTimeUpdate() {
      if (this.currentPlayer) {
        this.currentTimeRaw =
          this.currentPlayer.currentTime - this.frameDuration
      } else {
        this.currentTimeRaw = 0 + this.frameDuration
      }
      this.$emit(
        'frame-update',
        Math.round(this.currentTimeRaw / this.frameDuration)
      )
    },

    switchPlayers() {
      const nextIndex = this.getNextIndex(this.currentIndex)
      const nextEntity = this.entities[nextIndex]
      this.tmpPlayer = this.currentPlayer
      this.currentPlayer = this.nextPlayer
      this.nextPlayer = this.tmpPlayer
      if (nextEntity) {
        this.nextPlayer.src = this.getMoviePath(nextEntity)
      }
      this.resetHeight()
      const rate = this.$options.rate || 1
      this.setSpeed(rate)
    },

    updateTime(time) {
      const frameNumber = Math.round(time / this.frameDuration)
      this.$emit('frame-update', frameNumber)
    },

    updateMaxDuration() {
      if (this.currentPlayer) {
        this.$emit('max-duration-update', this.currentPlayer.duration)
      }
    },

    getSpeed(rate) {
      return this.currentPlayer ? this.currentPlayer.playbackRate : 0
    },

    setSpeed(rate) {
      this.$options.rate = rate
      if (this.currentPlayer) this.currentPlayer.playbackRate = rate
      if (this.nextPlayer) this.nextPlayer.playbackRate = rate
    },

    getNaturalDimensions() {
      return {
        height: this.currentPlayer.videoHeight,
        width: this.currentPlayer.videoWidth
      }
    }
  },

  watch: {
    isHd() {
      if (this.currentPlayer) {
        this.reloadCurrentEntity()
        if (this.isPlaying) this.play()
      }
    },

    entities() {
      if (this.entities.length > 0) {
        this.loadEntity(0)
        this.pause()
        this._setCurrentTime(0)

        const entity = this.entities[this.currentIndex]
        if (entity && !entity.preview_file_id) this.loadNextEntity()
      }
      setTimeout(() => {
        this.resetHeight()
      }, 300)
    },

    currentPreviewIndex() {
      if (!this.isPlaying) {
        const silent = true
        this.setCurrentTimeRaw(0)
        this.reloadCurrentEntity(silent)
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.video-wrapper {
  height: 100%;

  video {
    margin: auto;
  }
}

.container {
  max-height: 100%;
}
</style>
