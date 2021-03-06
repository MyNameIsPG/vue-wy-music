<template>
  <div>
    <div class="large-player" v-show="openedPlayer">
      <div v-if="song">
        <div class="background">
          <img width="100%" height="100%" :src="song.picUrl" />
        </div>
        <div class="top">
          <div class="back" @click="close">
            <i class="icon iconfont icondown-xiangxia1"></i>
          </div>
          <h1 class="title">{{song.name}}</h1>
          <h2 class="subtitle">{{song.author}}</h2>
        </div>
        <div
          class="middle"
          @touchstart.prevent="middleTouchStart"
          @touchmove.prevent="middleTouchMove"
          @touchend="middleTouchEnd"
        >
          <div class="middle-l" ref="middleL">
            <div class="cd-wrapper" ref="cdWrapper">
              <div class="cd" ref="imageWrapper">
                <img class="image" :class="isPlay ? 'rotation' : ''" :src="song.picUrl" />
              </div>
            </div>
            <div class="playing-lyric-wrapper">
              <div class="playing-lyric">{{playingLyric}}</div>
            </div>
          </div>
          <div class="middle-r" ref="lyricList">
            <div class="lyric-wrapper" v-if="currentLyric && currentLyric.lines">
              <Scroll
                ref="lyricSidebar"
                class="lyric-wrapper-bar"
                :before-scroll="beforeScroll"
                :listen-scroll="listenScroll"
                :data="currentLyric && currentLyric.lines"
              >
                <div class="pure-music">
                  <p
                    v-for="(item,index) in currentLyric.lines"
                    :key="index"
                    :class="{'current':currentLineNum===index}"
                  >{{item.txt}}</p>
                </div>
              </Scroll>
            </div>
          </div>
        </div>
        <div class="bottom">
          <div class="dot-wrapper">
            <span class="dot" :class="{'active':currentShow==='cd'}"></span>
            <span class="dot" :class="{'active':currentShow==='lyric'}"></span>
          </div>
          <div class="progress-wrapper">
            <span class="time time-l">{{this.currentTime}}</span>
            <div class="progress-bar-wrapper">
              <div class="progress-bar">
                <div class="bar-inner">
                  <div class="progress"></div>
                  <div class="progress-btn-wrapper">
                    <div class="progress-btn"></div>
                  </div>
                </div>
              </div>
            </div>
            <span class="time time-r">{{this.duration}}</span>
          </div>
          <div class="operators">
            <div class="icon i-left">
              <i class="icon iconfont iconicon-test9" @click="modeBtn"></i>
            </div>
            <div class="icon i-left" @click="prev">
              <i class="icon iconfont iconicon-test3"></i>
            </div>
            <div class="icon i-center" @click="playAndSuspend">
              <i class="icon iconfont" :class="isPlayClass"></i>
            </div>
            <div class="icon i-right" @click="next">
              <i class="icon iconfont iconicon-test7"></i>
            </div>
            <div class="icon i-right">
              <i class="icon iconfont iconlove"></i>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="small-player" @click="open" v-show="smallPlayer">
      <div class="left" v-if="song">
        <div class="player-bg">
          <img class="rotation" :src="song.picUrl" alt />
        </div>
        <div class="player-info">
          <h4>{{song.name}}</h4>
          <h5>{{song.author}}</h5>
        </div>
      </div>
      <div class="right" v-if="song">
        <span class="player">
          <i class="icon iconfont iconzanting"></i>
        </span>
        <span class="list">
          <i class="icon iconfont iconliebiao"></i>
        </span>
      </div>
    </div>
    <audio
      ref="audio"
      :autoplay="autoplay"
      controls="controls"
      @timeupdate="updateTime"
      style="opacity: 0;"
    ></audio>
  </div>
</template>

<script>
import { mapGetters, mapMutations } from "vuex";
import { reqSongUrl, reqLyric } from "@/api";
import { secondTime } from "@/utils/common";
import Lyric from "@/utils/lyric-parser";
import Scroll from "@/components/scroll/scroll";
let elementStyle = document.createElement("div").style;
let vendor = (() => {
  let transformNames = {
    webkit: "webkitTransform",
    Moz: "MozTransform",
    O: "OTransform",
    ms: "msTransform",
    standard: "transform"
  };

  for (let key in transformNames) {
    if (elementStyle[transformNames[key]] !== undefined) {
      return key;
    }
  }

  return false;
})();
export default {
  //mixins: [playerMixin],
  components: {
    Scroll
  },
  data() {
    return {
      beforeScroll: true,
      listenScroll: true,
      autoplay: false, //是否播放
      isPlay: true,
      current: 0,
      duration: 0,
      currentTime: 0,
      currentLyric: null,
      lyric: null,
      currentLineNum: 0,
      currentShow: "cd",
      playingLyric: ""
    };
  },
  computed: {
    isPlayClass: function() {
      return !this.isPlay ? "iconicon-test2" : "iconicon-test1";
    },
    ...mapGetters({
      song: "song",
      openedPlayer: "openedPlayer",
      smallPlayer: "smallPlayer",
      currentIndex: "currentIndex"
    })
  },
  created() {
    this.touch = {};
  },
  mounted() {
    if (this.openedPlayer) {
      this.play(this.song.id);
      this.lyricFn(this.song.id);
    }
  },
  methods: {
    // 播放和暂停按钮
    playAndSuspend() {
      this.isPlay = !this.isPlay;
      if (this.isPlay) {
        this.$refs.audio.play();
      } else {
        this.$refs.audio.pause();
      }
    },
    // 上一首
    prev() {
      if (this.current > 0) {
        this.current--;
      } else {
        this.current = this.song.order.length;
      }
      this.$store.dispatch("app/currentIndexData", this.current);
      this.switch();
    },
    // 下一首
    next() {
      if (this.current <= this.song.order.length - 1) {
        this.current++;
      } else {
        this.current = 0;
      }
      this.$store.dispatch("app/currentIndexData", this.current);
      this.switch();
    },
    // 切换调用的方法
    switch() {
      let data = {
        id: this.song.order[this.current].id,
        name: this.song.order[this.current].name,
        picUrl: this.song.order[this.current].picUrl,
        author: this.song.order[this.current].author,
        random: this.song.random,
        order: this.song.order
      };
      this.$store.dispatch("app/songData", data);
    },
    // 播放模式
    modeBtn() {},
    // 返回按钮
    close() {
      this.$store.dispatch("app/toggleOpenedPlayer");
      this.$store.dispatch("app/toggleSmallPlayer", true);
    },
    open() {
      this.$store.dispatch("app/toggleOpenedPlayer");
      this.$store.dispatch("app/toggleSmallPlayer", false);
    },
    // 播放事件
    play: async function(id) {
      let values = { id: id };
      const req = await reqSongUrl(values);
      if (req.data.code == 200) {
        this.current = this.currentIndex;
        this.$refs.audio.src = req.data.data[0].url;
        this.autoplay = true;
        this.$refs.audio.oncanplay = () => {
          this.duration = secondTime(this.$refs.audio.duration);
        };
      }
    },
    updateTime(e) {
      this.currentTime = secondTime(e.target.currentTime);
    },
    lyricFn: async function(id) {
      let values = { id: id };
      const req = await reqLyric(values);
      if (req.data.code == 200) {
        this.lyric = req.data.lrc.lyric;
        this.$nextTick(() => {
          this.currentLyric = new Lyric(this.lyric, this.handleLyric);
          this.currentLyric.play();
        });
      }
    },
    handleLyric({ lineNum, txt }) {
      this.currentLineNum = lineNum;
      this.playingLyric = txt;
      if (lineNum > 5) {
        let lineEl = this.$refs.lyricSidebar[lineNum - 5];
        this.$refs.lyricSidebar.scrollToElement(lineEl, 1000);
      } else {
        this.$refs.lyricSidebar.scrollToElement(0, 0, 1000);
      }
    },
    middleTouchStart(e) {
      this.touch.initiated = true;
      // 用来判断是否是一次移动
      this.touch.moved = false;
      const touch = e.touches[0];
      this.touch.startX = touch.pageX;
      this.touch.startY = touch.pageY;
    },
    middleTouchMove(e) {
      if (!this.touch.initiated) {
        return;
      }
      const touch = e.touches[0];
      const deltaX = touch.pageX - this.touch.startX;
      const deltaY = touch.pageY - this.touch.startY;
      const transform = this.prefixStyle("transform");
      const transitionDuration = this.prefixStyle("transitionDuration");
      if (Math.abs(deltaY) > Math.abs(deltaX)) {
        return;
      }
      if (!this.touch.moved) {
        this.touch.moved = true;
      }
      const left = this.currentShow === "cd" ? 0 : -window.innerWidth;
      const offsetWidth = Math.min(
        0,
        Math.max(-window.innerWidth, left + deltaX)
      );
      this.touch.percent = Math.abs(offsetWidth / window.innerWidth);
      this.$refs.lyricList.style[
        transform
      ] = `translate3d(${offsetWidth}px,0,0)`;
      this.$refs.lyricList.style[transitionDuration] = 0;
      this.$refs.middleL.style.opacity = 1 - this.touch.percent;
      this.$refs.middleL.style[transitionDuration] = 0;
    },
    middleTouchEnd(e) {
      if (!this.touch.moved) {
        return;
      }
      const transform = this.prefixStyle("transform");
      const transitionDuration = this.prefixStyle("transitionDuration");
      let offsetWidth;
      let opacity;
      if (this.currentShow === "cd") {
        if (this.touch.percent > 0.1) {
          offsetWidth = -window.innerWidth;
          opacity = 0;
          this.currentShow = "lyric";
        } else {
          offsetWidth = 0;
          opacity = 1;
        }
      } else {
        if (this.touch.percent < 0.9) {
          offsetWidth = 0;
          this.currentShow = "cd";
          opacity = 1;
        } else {
          offsetWidth = -window.innerWidth;
          opacity = 0;
        }
      }
      const time = 300;
      this.$refs.lyricList.style[
        transform
      ] = `translate3d(${offsetWidth}px,0,0)`;
      this.$refs.lyricList.style[transitionDuration] = `${time}ms`;
      this.$refs.middleL.style.opacity = opacity;
      this.$refs.middleL.style[transitionDuration] = `${time}ms`;
      this.touch.initiated = false;
    },
    prefixStyle(style) {
      if (vendor === false) {
        return false;
      }

      if (vendor === "standard") {
        return style;
      }

      return vendor + style.charAt(0).toUpperCase() + style.substr(1);
    }
  },
  watch: {
    // 深度监听歌曲id的变化
    song: {
      handler(n, o) {
        if (o) {
          if (n.id !== o.id) {
            this.play(n.id);
            this.lyricFn(n.id);
          }
        } else {
          this.play(n.id);
          this.lyricFn(n.id);
        }
      },
      deep: true
    }
  }
};
</script>

<style lang="stylus" scoped>
@import '~@/assets/css/variable.styl';

.large-player {
  position: fixed;
  left: 0px;
  top: 0px;
  width: 100%;
  height: 100%;
  background: #222;
  z-index: 11;

  .background {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
    opacity: 0.6;
    filter: blur(20px);
  }

  .top {
    position: relative;
    margin-bottom: 25px;

    .back {
      position: absolute;
      top: 0;
      left: 6px;
      z-index: 50;

      i {
        display: block;
        padding: 9px;
        font-size: 22px;
        color: #fff;
      }
    }

    .title {
      width: 70%;
      margin: 0 auto;
      line-height: 40px;
      text-align: center;
      no-wrap();
      font-size: 18px;
      color: #fff;
    }

    .subtitle {
      line-height: 20px;
      text-align: center;
      font-size: 14px;
      color: #fff;
    }
  }

  .middle {
    position: fixed;
    width: 100%;
    top: 80px;
    bottom: 170px;
    white-space: nowrap;
    font-size: 0;

    .middle-l {
      display: inline-block;
      vertical-align: top;
      position: relative;
      width: 100%;
      height: 0;
      padding-top: 80%;

      .cd-wrapper {
        position: absolute;
        left: 10%;
        top: 0;
        width: 80%;
        box-sizing: border-box;
        height: 100%;

        .cd {
          width: 100%;
          height: 100%;
          border-radius: 50%;

          .image {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            box-sizing: border-box;
            border-radius: 50%;
            border: 10px solid rgba(255, 255, 255, 0.1);
          }

          @keyframes rotation {
            from {
              -webkit-transform: rotate(0deg);
            }

            to {
              -webkit-transform: rotate(360deg);
            }
          }

          .rotation {
            -webkit-transform: rotate(360deg);
            animation: rotation 15s linear infinite;
            -moz-animation: rotation 15s linear infinite;
            -webkit-animation: rotation 15s linear infinite;
            -o-animation: rotation 15s linear infinite;
          }

          .play {
            animation: rotate 20s linear infinite;
          }
        }
      }

      .playing-lyric-wrapper {
        width: 80%;
        margin: 30px auto 0 auto;
        overflow: hidden;
        text-align: center;

        .playing-lyric {
          height: 20px;
          line-height: 20px;
          font-size: 14px;
          color: #ffffff;
        }
      }
    }

    .middle-r {
      display: inline-block;
      vertical-align: top;
      width: 100%;
      height: 100%;
      overflow: hidden;

      .lyric-wrapper {
        width: 80%;
        margin: 0 auto;
        overflow: hidden;
        text-align: center;
        height: 100%;

        .lyric-wrapper-bar {
          height: 100%;
          overflow: hidden;
        }

        .text {
          line-height: 32px;
          color: rgba(255, 255, 255, 0.5);
          font-size: 14px;
        }

        .current {
          color: #fff;
        }

        .pure-music {
          line-height: 32px;
          color: rgba(255, 255, 255, 0.5);
          font-size: 14px;
        }
      }
    }
  }

  .bottom {
    position: absolute;
    bottom: 50px;
    width: 100%;

    .dot-wrapper {
      text-align: center;
      font-size: 0;

      .dot {
        display: inline-block;
        vertical-align: middle;
        margin: 0 4px;
        width: 8px;
        height: 8px;
        border-radius: 50%;
        background: rgba(255, 255, 255, 0.5);

        &.active {
          width: 20px;
          border-radius: 5px;
          background: rgba(255, 255, 255, 0.8);
        }
      }
    }

    .progress-wrapper {
      display: flex;
      align-items: center;
      width: 80%;
      margin: 0px auto;
      padding: 10px 0;

      .progress-bar-wrapper {
        width: 100%;

        .progress-bar {
          height: 30px;

          .bar-inner {
            position: relative;
            top: 13px;
            height: 4px;
            background: rgba(0, 0, 0, 0.3);

            .progress {
              position: absolute;
              height: 100%;
              background: #ffcd32;
            }

            .progress-btn-wrapper {
              position: absolute;
              left: -8px;
              top: -13px;
              width: 30px;
              height: 30px;

              .progress-btn {
                position: relative;
                top: 7px;
                left: 7px;
                box-sizing: border-box;
                width: 16px;
                height: 16px;
                border: 3px solid #fff;
                border-radius: 50%;
                background: #ffcd32;
              }
            }
          }
        }
      }

      .time {
        color: #fff;
        font-size: 12px;
        flex: 0 0 30px;
        line-height: 30px;
        width: 30px;

        &.time-l {
          text-align: left;
        }

        &.time-r {
          text-align: right;
        }
      }
    }

    .operators {
      display: flex;
      align-items: center;

      .icon {
        flex: 1;
        color: #ffcd32;

        i {
          font-size: 30px;
        }
      }

      .i-left {
        text-align: right;
      }

      .i-center {
        padding: 0 20px;
        text-align: center;

        i {
          font-size: 40px;
        }
      }

      .i-right {
        text-align: left;
      }

      .icon-favorite {
        color: #d93f30;
      }
    }
  }
}

.small-player {
  position: fixed;
  bottom: 0px;
  left: 0px;
  width: 100%;
  height: 60px;
  overflow: hidden;
  border-top: 1px solid #ebebeb;
  padding: 0 10px;
  box-sizing: border-box;
  background: #ffffff;
  z-index: 11;

  .left {
    height: 100%;
    float: left;
    display: flex;
    align-items: center;

    .player-bg {
      width: 50px;
      height: 50px;
      overflow: hidden;
      margin-right: 10px;

      img {
        width: 100%;
        height: 100%;
        border-radius: 50%;
      }

      @keyframes rotation {
        from {
          -webkit-transform: rotate(0deg);
        }

        to {
          -webkit-transform: rotate(360deg);
        }
      }

      .rotation {
        -webkit-transform: rotate(360deg);
        animation: rotation 15s linear infinite;
        -moz-animation: rotation 15s linear infinite;
        -webkit-animation: rotation 15s linear infinite;
        -o-animation: rotation 15s linear infinite;
      }
    }

    .player-info {
      h4 {
        color: #353535;
        font-size: 16px;
      }

      h5 {
        margin-top: 6px;
        color: #808080;
        font-size: 14px;
      }
    }
  }

  .right {
    height: 100%;
    float: right;
    display: flex;
    align-items: center;

    .player {
      i {
        color: #555555;
        font-size: 40px;
      }
    }

    .list {
      i {
        color: #555555;
        font-size: 40px;
      }
    }
  }
}
</style>