<template>
  <div>
    <div
      :class="['play-bar', lockName]"
      v-if="curSongInfo"
      @mouseenter="enterBar"
      @mouseleave="leaveBar"
    >
      <div class="fold">
        <!-- 播放条上面的小锁 -->
        <div class="fold-btn" @click="lockBar">
          <i class="iconfont icon-lock" :class="[locked ? 'active' : '']"></i>
        </div>
      </div>
      <!-- 给子组件传参： 进度条时长 -->
      <!-- 向子组件传递方法 ： 点击拖拽进度条，设置当前时间 -->
      <progress-line
        class="audioProgress"
        :progressWidth="audioProgressWidth"
        @setProgressLine="setAudioProgress"
      ></progress-line>
      <div class="w1200">
        <audio
          ref="audio"
          preload="auto"
          @canplay="canplaySong"
          @playing="playSong"
          @ended="endedSong"
          @error="errorSong"
          @timeupdate="updateSongTime"
          :src="curSongInfo.url"
        ></audio>
        <el-row class="play-bar-inside">
          <!-- 歌曲封面+名字+作者+时间 -->
          <el-col :span="7" class="bar-l">
            <router-link :to="{ path: '/song', query: { id: curSongInfo.id } }">
              <el-image :src="curSongInfo.album.picUrl" class="bar-cover-img">
                <div slot="placeholder" class="image-slot">
                  <i class="iconfont icon-placeholder"></i>
                </div>
              </el-image>
            </router-link>
            <div class="bar-name">
              <router-link
                :to="{ path: '/song', query: { id: curSongInfo.id } }"
                class="song_name"
                >{{ curSongInfo.name }}</router-link
              >
              <p>
                <router-link
                  :to="{ path: '/artist/singer', query: { id: author.id } }"
                  class="song_author"
                  v-for="(author, k) in curSongInfo.singer"
                  :key="author.name"
                  >{{
                    k !== 0 ? " / " + author.name : author.name
                  }}</router-link
                >
              </p>
            </div>
            <div class="bar-time">
              <span>{{ $utils.formatSongTime(currentTime * 1000) }}</span> /
              {{ curSongInfo.duration }}
            </div>
          </el-col>
          <!-- 上下首和暂停按键 -->
          <el-col :span="9" class="bar-m">
            <div class="bar-control">
              <i
                class="iconfont icon-audio-prev"
                @click.stop="audioHandler('prev')"
              ></i>
              <i
                :class="['iconfont', playIcon]"
                @click.stop="audioHandler('play')"
              ></i>
              <i
                class="iconfont icon-audio-next"
                title="下一首"
                @click.stop="audioHandler('next')"
              ></i>
            </div>
          </el-col>
          <!-- 音量，循环播放，歌词显示+歌单列表 -->
          <el-col :span="8" class="bar-r">
            <div class="bar-oper">
              <!-- 音量 -->
              <div class="volumn-main">
                <i
                  :class="['iconfont', mutedIcon]"
                  title="音量"
                  @click.stop="volumeHandler"
                ></i>
                <!-- 给子组件传参 -->
                <progress-line
                  class="volumeLine"
                  :progressWidth="volumnProgressWidth"
                  @setProgressLine="setvolumeProgress"
                >
                </progress-line>
              </div>
              <!-- 单曲+循环+随机播放 -->
              <i
                class="iconfont"
                :class="modeIcon.className"
                :title="modeIcon.title"
                @click.stop="changePlayMode"
              ></i>
              <!-- 歌单 -->
              <div class="popver" v-clickoutside="popverClose">
                <!-- 歌词 -->
                <div class="lyric">
                  <span class="lyric-btn" title="歌词" @click="lyricsHandle"
                    >词</span
                  >
                </div>
                <!-- 歌单列表 -->
                <div class="playlist" @click="playlistHandle">
                  <i class="iconfont icon-playlist" title="播放列表"></i>
                  <div class="playlist-num">
                    {{ 99 > playList.length ? playList.length : "99+" }}
                  </div>
                </div>

                <!-- 绝对定位的歌词和歌单列表 -->
                <div class="lyrics-container" v-show="lyricsVisible">
                  <h3 class="lyrics-header">
                    <span>歌词</span>
                    <i class="iconfont icon-closed" @click="lyricsHandle"></i>
                  </h3>
                  <lyrics
                    :sId="curSongInfo.id"
                    :currentTime="currentTime"
                  ></lyrics>
                </div>
                <!-- 播放列表盒子 -->
                <div class="playlist-container" v-show="playlistVisible">
                  <h3 class="playlist-header">
                    <span
                      >播放列表<em>(共{{ playList.length }}首)</em></span
                    >
                    <div class="delSonglist" @click="clearSonglist">
                      <i class="iconfont icon-del" title="清空列表"></i>清空列表
                    </div>
                  </h3>
                  <song-list
                    :songList="playList"
                    :typeSize="'mini'"
                    :height="400"
                    :isScroll="true"
                    :isShowTips="false"
                  ></song-list>
                </div>
              </div>
            </div>
          </el-col>
        </el-row>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapMutations } from "vuex";
import ProgressLine from "@/components/common/Play-bar-Progress";
import SongList from "@/components/common/Play-bar-SongList";
import Lyrics from "@components/common/Play-bar-Lyrics.vue";
export default {
  components: {
    ProgressLine,
    SongList,
    Lyrics,
  },
  data() {
    return {
      isLock: false,
      locked: false,
      lockName: "active",
      initAudioReady: false, // 初始化音频准备
      currentTime: 0, // 音频当前时长
      totalTime: 0, // 音频总时长
      playMode: 0, /// 播放模式 ， 0循环播放 ， 1 单曲播放。 2随机播放
      isMuted: false, // 是否静音
      volumeNum: 0.5, // 音量值（0-1）
      oldVolumn: 0, // 取消静音时，设置保留的上一次的音量值
      playlistVisible: false, // 歌单列表是否显现的状态值
      timer: null,
      lyricsVisible: false,
    };
  },
  mounted() {
    this.leaveBar(this);
  },
  computed: {
    ...mapGetters(["playIndex", "playList", "isPlayed", "playListTips"]),
    curSongInfo() {
      // console.log(this.playList[this.playIndex]);
      return this.playList[this.playIndex];
    },
    // 音频进度条长度
    audioProgressWidth() {
      return (this.currentTime / this.totalTime) * 100 + "%";
    },

    // 播放暂停按钮
    playIcon() {
      return !this.isPlayed ? "icon-audio-play" : "icon-audio-pause";
    },
    // 播放模式
    modeIcon() {
      const params = [
        {
          className: "icon-loop",
          title: "循环模式",
        },
        {
          className: "icon-single-cycle",
          title: "单曲循环",
        },
        {
          className: "icon-shuffle",
          title: "随机播放",
        },
      ];
      return params[this.playMode];
    },
    // 是否静音
    mutedIcon() {
      return this.isMuted ? "icon-volume-active" : "icon-volume";
    },
    volumnProgressWidth() {
      return (this.volumeNum / 1) * 100 + "%";
    },
  },
  methods: {
    ...mapMutations({
      setPlayStatus: "SET_PLAYSTATUS",
      setPlayIndex: "SET_PLAYINDEX",
      setPlayList: "SET_PLAYLIST",
    }),
    enterBar() {
      clearTimeout(this.timer);
      this.lockName = "active";
    },
    leaveBar(e) {
      // 点击锁住按钮，会触发mouseleave 事件 此时的e的值是 undefined  而正常通过鼠标移出的时候 e是个对象
      const self = this;
      if (!self.isLock && !self.locked) {
        clearTimeout(self.timer);
        self.timer = setTimeout(() => {
          self.lockName = self.isLock ? "active" : "";
        }, 3000);
      }
    },
    lockBar() {
      this.locked = !this.locked;
      this.isLock = this.locked;
      this.leaveBar();
    },

    // 点击拖拽进度条，设置当前时间
    setAudioProgress(params) {
      this.initAudioReady = false;
      console.log(params);
      this.currentTime = params.val * this.totalTime;
      // console.log(this.currentTime);
      // 拖拽时，不实时改变音频的时间
      if (params.flag) {
        this.$refs.audio.currentTime = params.val * this.totalTime;
      }
    },
    // 刷新页面时，音频准备就绪
    canplaySong(e) {
      this.initAudioReady = true;
    },
    // 音频播放时候，初始化状态，获取音频总时长
    playSong(e) {
      this.initAudioReady = true;
      this.setPlayStatus(true);
      console.log(e);
      this.totalTime = e.target.duration;
    },
    // 音频播放结束，自动播放下一首
    endedSong(e) {
      if (this.playMode === 1) {
        this.loopSong();
      } else {
        this.changeSong("next");
      }
    },
    // 音频加载失败
    errorSong(e) {
      this.initAudioReady = false;
      this.setPlayStatus(false);
      this.$msg.error("音频加载失败");
    },
    // 监听音频时间，实时 更新当前播放时间
    updateSongTime(e) {
      if (!this.initAudioReady) {
        return;
      }
      this.currentTime = e.target.currentTime;
      this.$bus.$emit("currTime", this.currentTime);
    },
    // 音量静音
    // 单曲循环歌曲
    loopSong() {
      this.$refs.audio.currentTime = 0;
      this.$refs.audio.play();
      this.setPlayStatus(true);
    },
    // 手动 切换歌曲
    changeSong(type) {
      //type: pre/next  上一首/下一首
      if (this.playList.lenght !== 1) {
        let index = this.playIndex;
        if (this.playMode === 2) {
          // 随机模式
          index = Math.floor(Math.random() * this.playList.length - 1) + 1;
        } else {
          if (type === "prev") {
            index = index === 0 ? this.playList.length - 1 : --index;
          } else {
            index = index >= this.playList.length - 1 ? 0 : ++index;
          }
        }
        // 设置当前歌曲的播放状态为false
        this.initAudioReady = false;
        this.setPlayStatus(false);
        this.setPlayIndex(index);
      } else {
        this.loopSong(); // 播放列表只有一首歌；单曲循环
      }
    },

    // ---------------- 音频播放/暂停/上一首/下一首事件
    audioHandler(type) {
      if (type === "play") {
        this.setPlayStatus(!this.isPlayed);
        this.setPlayIndex(this.playIndex);
      } else {
        this.changeSong(type);
      }
    },
    // 切换播放模式
    changePlayMode() {
      this.playMode = this.playMode >= 2 ? 0 : this.playMode + 1;
    },
    // ------------------------音量静音 及 取消操作
    volumeHandler() {
      this.isMuted = this.$refs.audio.muted = this.isMuted ? 0 : 1;
      this.isMuted && (this.oldVolume = this.volumeNum);
      this.volumeNum = this.isMuted ? 0 : this.oldVolume;
    },
    // 点击拖拽音量条，设置当前音量
    setvolumeProgress(params) {
      this.volumeNum = params.val;
      this.oldVolume = params.val;
      this.$refs.audio.volume = params.val;
      this.isMuted = this.$refs.audio.muted = params.val ? 0 : 1;
    },
    // ------------------播放列表盒子
    //  点击框外区域 不显示
    popverClose() {
      // 歌词不显示
      this.lyricsVisible = this.playlistVisible = this.isLock = false;
      // 播放列表不显示
      this.leaveBar();
    },
    // 点击显示歌词
    lyricsHandle() {
      this.lyricsVisible = !this.lyricsVisible;
      this.playlistVisible = false;
      this.isLock = this.lyricsVisible;
    },
    clearSonglist() {
      window.localStorage.removeItem("playList");
      this.setPlayList([]);
      this.setPlayStatus(false);
      this.setPlayIndex(0);
    },
    playlistHandle() {
      this.playlistVisible = !this.playlistVisible;
      this.lyricsVisible = false;
      this.isLock = this.playlistVisible;
    },
  },
  watch: {
    curSongInfo(newSong, oldSong) {
      if (!newSong || (oldSong && newSong.id === oldSong.id)) {
        return;
      }
      // 当前播放歌曲变化的时候  重置状态及当前播放的时长
      this.initAudioReady = false;
      this.currentTime = 0;
      this.$nextTick(() => {
        const audio = this.$refs.audio;
        if (audio) {
          audio.play();
        }
      });
    },
    isPlayed(playing) {
      // 等待音频加载成功完成后播放
      if (!this.initAudioReady) {
        return;
      }
      this.$nextTick(() => {
        const audio = this.$refs.audio;
        if (audio) {
          playing ? audio.play() : audio.pause();
        }
      });
    },
  },
  destroyed() {
    clearTimeout(this.timer);
  },
};
</script>

<style lang="less" scoped>
.play-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  z-index: 5;
  width: 100%;
  height: 70px;
  background: #fff;
  box-shadow: 0 5px 40px -1px rgba(2, 10, 18, 0.1);
  transition: all 0.4s ease-out;
  transform: translateY(100%);

  &.active {
    transform: translateY(0);
  }
}
.fold {
  position: absolute;
  top: -30px;
  left: 0;
  z-index: 3;
  width: 100%;
  height: 30px;

  .fold-btn {
    position: absolute;
    right: 200px;
    display: inline-block;
    width: 60px;
    height: 30px;
    line-height: 30px;
    text-align: center;
    border-radius: 50px 50px 0px 0px;
    background: #fff;
    box-shadow: 0 5px 40px -1px rgba(2, 10, 18, 0.1);
    cursor: pointer;

    .active {
      color: #ff641e;
    }
  }
}
.audioProgress {
  position: absolute;
  top: -6px;
  z-index: 9;
}
.play-bar-inside {
  .bar-l {
    display: flex;
    padding: 10px 0;
    align-items: center;
    justify-content: center;
    .bar-cover-img {
      width: 50px;
      height: 50px;
      border-radius: 4px;
      box-shadow: 0 0 15px 5px rgba(#ff641e, 0.15);
    }
    .bar-name {
      flex: 1;
      font-size: 0;
      padding: 0 10px 6px 30px;
      overflow: hidden;

      .song_name {
        display: block;
        padding-bottom: 20px;
        line-height: 12px;
        font-size: 12px;
        font-weight: bold;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      p {
        line-height: 0;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      .song_author {
        line-height: 12px;
        font-size: 12px;
        color: #999;
      }
    }
    .bar-time {
      width: 115px;
      line-height: 50px;
      font-size: 14px;
      color: #606266;
      text-align: center;
    }
  }

  .bar-control {
    display: flex;
    padding: 10px 0;
    line-height: 1;
    align-items: center;
    justify-content: center;

    .iconfont {
      margin: 0 20px;
      font-size: 30px;
      cursor: pointer;
      color: #666;
    }

    .icon-audio-play,
    .icon-audio-pause {
      font-size: 50px;
      font-weight: bold;
      color: #ff641e;
    }
  }
  .bar-oper {
    position: relative;
    display: flex;
    font-size: 0;
    line-height: 50px;
    padding-left: 30px;
    align-items: center;
    justify-content: center;

    .volumn-main {
      // width: 300px;
      padding: 10px 0;
      display: flex;
      align-items: center;
      .volumeLine {
        width: 200px;
        margin-left: -70px;
        display: inline-block;
        flex: 1;
      }
      .iconfont {
        // margin-left: 40px;
        font-size: 24px;
        cursor: pointer;
        flex: 1;
      }
    }

    .iconfont {
      margin-left: 20px;
      font-size: 24px;
      cursor: pointer;
    }

    .popver {
      position: relative;
      .lyric {
        display: inline-block;
        padding: 26px 3px;
        margin-left: 20px;
        font-size: 18px;
        line-height: 18px;
        vertical-align: top;
        color: #999;
        cursor: pointer;
      }

      .playlist {
        position: relative;
        display: inline-block;
        line-height: 70px;
        color: #999;
        cursor: pointer;

        .playlist-num {
          position: absolute;
          top: 25%;
          left: 90%;
          font-size: 12px;
          line-height: 12px;
          color: #999;
        }
      }

      .playlist-container {
        position: absolute;
        left: -250px;
        bottom: 75px;
        width: 500px;
        padding: 20px;
        border: 1px solid #ebeef5;
        border-radius: 4px 4px 0 0;
        font-size: 14px;
        box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
        background: #fff;
        .playlist-header {
          display: flex;
          line-height: 40px;
          span {
            display: inline-block;
            flex: 1;
          }

          em {
            display: inline-block;
            padding-left: 10px;
            font-size: 12px;
            line-height: 14px;
            font-style: normal;
            font-weight: normal;
            color: #666;
            vertical-align: baseline;
          }

          .delSonglist {
            font-size: 14px;
            font-weight: normal;
            cursor: pointer;
          }

          .icon-del {
            margin-right: 5px;
            font-size: 16px;
            color: #333;
          }
        }
      }
    }
  }
}
.lyrics-container {
  position: absolute;
  left: -200px;
  bottom: 75px;
  width: 400px;
  padding: 20px;
  height: 430px;
  border: 1px solid #ebeef5;
  border-radius: 4px 4px 0 0;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  background: #fff;

  .lyrics-header {
    display: flex;
    padding: 0 0 24px;
    line-height: 16px;
    font-weight: 500;
    font-size: 16px;

    span {
      display: block;
      flex: 1;
    }

    .icon-closed {
      font-size: 20px;
    }
  }
}
.playlist-container {
  position: absolute;
  left: -250px;
  bottom: 75px;
  width: 500px;
  padding: 20px;
  border: 1px solid #ebeef5;
  border-radius: 4px 4px 0 0;
  font-size: 14px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  background: #fff;
}
</style>
