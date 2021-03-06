<template>
  <div class="page" :style="{ height: height + 'px' }">
    <div class="player">
      <button class="prev-song" @click="prevSong" :disabled="!playerState.ready">
        <i class="iconfont icon-music-prev"></i>
      </button>
      <div class="album-box">
        <div class="circle-progress">
          <svg viewBox="0 0 100 100">
            <path class="circle-progress-circle-track" 
                  stroke="rgba(197, 197, 197, 0.4)" 
                  fill="none"
                  :d="trackPath" 
                  :stroke-width="relativeStrokeWidth" >
            </path>
            <path class="circle-progress-circle-path" 
                  stroke="rgba(190, 190, 190, 0.7)" 
                  stroke-linecap="bevel" 
                  fill="none" 
                  :d="trackPath" 
                  :stroke-width="relativeStrokeWidth" 
                  :style="circlePathStyle">
            </path>
          </svg>
        </div>
        <div class="song-bg-box" :class="{ 'playing': playerState.playing }">
          <img :src="currentSongPicUrl">
        </div>
        <div class="toggle-box">
          <transition name="module" mode="out-in">
            <button class="toggle-btn" @click="togglePlay" v-if="playerState.playing" :disabled="!playerState.ready">
              <i class="iconfont icon-music-pause"></i>
            </button>
            <button class="toggle-btn" @click="togglePlay" v-else :disabled="!playerState.ready">
              <i class="iconfont icon-music-play"></i>
            </button>
          </transition>
        </div>
        <div class="toggle-muted">
          <button class="muted-btn" @click="toggleMuted" :disabled="!playerState.ready">
            <i class="iconfont" :class="[playerState.muted ? 'icon-music-muted' : 'icon-music-volume']"></i>
          </button>
        </div>
      </div>
      <button class="next-song" @click="nextSong" :disabled="!playerState.ready">
        <i class="iconfont icon-music-next"></i>
      </button>
    </div>
    <div class="song-info">
      <h3 class="name">
        <span v-if="currentSong">
          <span>{{ currentSong.name }}</span>
          <span> By </span>
          <span :key="index" v-for="(artist, index) in currentSong.artists">{{ artist.name }}</span>
          <span> / </span>
          <span>{{ currentSong.album.name || 'unknow' }}</span>
        </span>
        <span v-else>Kind words are the music of the world.</span>
      </h3>
      <p class="lrc">
        <span v-if="currentSongLrc.fetching">歌词加载中...</span>
        <span v-else>
          <span v-if="!songLrcContent">暂无歌词</span>
          <span v-else>
            <span v-if="songLrcContent.version < 3">非滚动歌词，所以我就不显示了</span>
            <span v-else>
              <transition name="module" mode="out-in">
                <span class="lrc-text" :key="currentTimeLrc" v-text="currentTimeLrc"></span>
              </transition>
            </span>
          </span>
        </span>
      </p>
    </div>
  </div>
</template>

<script>
  import EventBus from '~/utils/event-bus'
  import { isBrowser } from '~/environment'

  export default {
    name: 'music',
    head() {
      return {
        title: `${this.langIsEn ? '' : this.$i18n.nav.music + ' | '}Music`
      }
    },
    data() {
      return {
        height: 0
      }
    },
    created() {
      if (this.$store.state.option.mobileLayout) {
        this.$router.back()
      }
    },
    mounted() {
      if (isBrowser) {
        this.updateScreenHeight()
        window.addEventListener('resize', this.updateScreenHeight)
      }
    },
    beforeDestroy() {
      window.removeEventListener('resize', this.updateScreenHeight)
    },
    computed: {
      langIsEn() {
        return this.$store.getters['option/langIsEn']
      },
      player() {
        return EventBus.player.player
      },
      playerState() {
        return EventBus.player.playerState
      },
      currentSong() {
        return EventBus.currentSong
      },
      currentSongLrc() {
        return EventBus.player.lrc
      },
      songLrcContent() {
        const lrc = this.currentSongLrc.data
        if (!lrc || lrc.nolyric) {
          return null
        } else {
          return lrc.lrc
        }
      },
      songLrcArr() {
        return this.songLrcContent.lyric.split('\n').map(timeSentence => {
          let time = /\[([^\[\]]*)\]/.exec(timeSentence)
          time = time && time.length && time[1]
          time = time && time.split(':').map(t => Number(t))
          time = time && time.length && time.length > 1 && time[0] * 60 + time[1]
          time = time || ''
          let sentence = /([^\]]+)$/.exec(timeSentence)
          sentence = sentence && sentence.length && sentence[1]
          sentence = sentence || ''
          return { time, sentence }
        }).filter(ts => ts.time)
      },
      currentTimeLrc() {
        const currentTime = this.playerState.seek
        if (!this.songLrcArr.length) {
          return '无滚动歌词'
        }
        const targetSentence = this.songLrcArr.find((ts, i, a) => {
          const next = a[i + 1]
          return ts.time <= currentTime && next && next.time > currentTime
        })
        return targetSentence ? targetSentence.sentence : '...'
      },
      currentSongPicUrl() {
        return EventBus.currentSongPicUrl
      },
      relativeStrokeWidth() {
        return (15 / 450 * 100).toFixed(1)
      },
      trackPath() {
        const radius = parseInt(50 - parseFloat(this.relativeStrokeWidth) / 2, 10)
        return `M 50 50 m 0 -${radius} a ${radius} ${radius} 0 1 1 0 ${radius * 2} a ${radius} ${radius} 0 1 1 0 -${radius * 2}`
      },
      perimeter() {
        const radius = 50 - parseFloat(this.relativeStrokeWidth) / 2
        const result  = 2 * Math.PI * radius
        return result
      },
      circlePathStyle() {
        const perimeter = this.perimeter
        return {
          strokeDasharray: `${perimeter}px,${perimeter}px`,
          strokeDashoffset: (1 - (this.playerState.progress) / 100) * perimeter + 'px',
          transition: 'stroke-dashoffset 0.6s ease 0s, stroke 0.6s ease'
        };
      }
    },
    methods: {
      updateScreenHeight(event) {
        const screenHeight = window.innerHeight
        const minHeight = 14 * 53
        if (screenHeight - 14 * 4 > minHeight) {
          this.height = screenHeight - (14 * 12)
        } else {
          this.height = minHeight
        }
      },
      checkPLayerState(action) {
        this.playerState.ready && action()
      },
      togglePlay() {
        this.checkPLayerState(this.player.togglePlay)
      },
      toggleMuted() {
        this.checkPLayerState(this.player.toggleMuted)
      },
      prevSong() {
        this.checkPLayerState(this.player.prevSong)
      },
      nextSong() {
        this.checkPLayerState(this.player.nextSong)
      },
      getSongLrc(song_id) {
        EventBus.getLrcFailure()
        this.$store.dispatch('loadMuiscSongLrc', song_id)
      }
    },
    watch: {
      currentSong() {
        const song = this.currentSong
        if (song && song.id) {
          this.getSongLrc(song.id)
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  .page {
    display: flex;
    justify-content: center;
    flex-direction: column;
    align-items: center;

    > .player {
      display: flex;
      justify-content: space-around;
      align-items: center;
      margin-bottom: 2em;
      width: 100%;

      > .prev-song,
      > .next-song {
        width: 3rem;

        &:hover {

          > .iconfont {
            color: $text;
          }
        }

        > .iconfont {
          font-size: 3em;
          color: $dividers;
        }
      }

      > .album-box {
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        width: 38rem;
        height: 38rem;
        opacity: .9;

        &:hover {
          opacity: 1;
        }

        @keyframes rotation {
          from { transform: rotate(0deg) }
          to { transform: rotate(360deg) }
        }

        > .song-bg-box {
          position: absolute;
          width: 100%;
          height: 100%;
          padding: 1rem;
          overflow: hidden;
          border-radius: 100%;
          animation: rotation 26s linear infinite;
          animation-play-state: paused;

          &.playing {
            animation-play-state: running;
          }

          > img {
            width: 100%;
            height: 100%;
            border-radius: 100%;
            background-color: darkgray;
            background-image: url('/images/music-bg.jpg');
            background-size: cover;
          }
        }

        > .toggle-box {
          z-index: 9;

          > .toggle-btn {
            width: 6rem;
            height: 6rem;
            line-height: 6rem;
            text-align: center;
            background-color: $module-bg;
            border-radius: 100%;
            opacity: .5;

            &:hover {
              opacity: .8;
              transform: scale(1.2);
            }

            > .iconfont {
              color: $text-reversal;
              font-size: 3em;
            }
          }
        }

        > .toggle-muted {
          position: absolute;
          bottom: 15%;

          > .muted-btn {

            > .iconfont {
              font-size: 2em;
              color: $module-hover-bg;
            }

            &:hover {

              > .iconfont {
                color: $module-bg;
              }
            }
          }
        }

        > .circle-progress {
          width: 100%;
          height: 100%;
          display: block;
          position: absolute;
        }
      }
    }

    > .song-info {
      text-align: center;
      
      > .name {
        margin-bottom: 1em;
      }

      > .lrc {

        @keyframes lrc-text {
          0% { color: $primary }
          33% { color: $red }
          66% { color: $accent }
        }

        .lrc-text {
          color: $primary;
          // animation: lrc-text 5s linear infinite;
        }
      }
    }
  }
</style>
