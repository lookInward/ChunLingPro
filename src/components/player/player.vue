<template>
    <div class="playerPage">
        <div class="player" id="player" v-show="playlist.length>0">
          <div class="fixed" v-show="fullScreen">
            <div class="normal-player" v-show="fullScreen">
                <h2 v-html="currentSong.name"></h2>
                         <div class="round">
                           <div class="image-round">
                    <img :src="currentSong.image" alt="">
                        </div>
                    <div class="back title-color" @click=back><a>返回</a></div>
                </div>
                <div class="detail">
                    <div class="title-album"><span class="title-color">专辑：</span><span v-text="currentSong.album"></span></div>
                    <div class="title-singer"><span class="title-color">歌手：</span><span v-text="currentSong.singer"></span></div>
                    <div class="lyric">
                        <div class="lyric-wrapper">
                            <div v-if="currentLyric" class="lyricInner" ref="lyricMove">
                                <p ref="lyricLine" class="text" :class="{'current':currentLineNum===index}" v-for="(line,index) in currentLyric.lines" :key="line" v-text="line.txt"></p>
                            </div>
                            <div v-else>
                                 <p><br><br></p>
                                <div>这首歌暂无歌词！</div>
                            </div>
                        </div>
                    </div>
                </div>
                <playlist ref="playlistOne"></playlist>
                <div class="controller">
                    <div class="mini-player" v-show="fullScreen">
                        <div class="mini-control iconfont">
                            <span @click="prev" class="smaller icon-kuaitui"></span>
                            <span @click="togglePlaying" :class="playIcon"></span>
                            <span @click="next" class="smaller icon-kuaijin"></span>
                        </div>
                        <div class="mini-img">
                            <img class="PC-Show" :src="currentSong.image" alt="">
                        </div>
                        <!-- 进度条 -->
                        <div class="process" @mouseup="doUp" @mousemove="doMove">
                            <div class="songname"><span v-text="currentSong.name"></span></div>
                            <div class="time">
                                <span class="progress-time" v-text="format(currentTime)"></span>
                                <span class="separate">/</span>
                                <span class="progress-time" v-text="format(currentSong.duration)"></span>
                            </div>
                            <div class="progress-bar" ref="progressBar" @click="progressClick">
                                <div class="bar-inner" ref="barInner">
                                    <div class="progress-ago" ref="progressAgo"></div>
                                    <div class="progress-btn-wrapper" ref="progressBtn" @mousedown="doDown"></div>
                                    <div class="progress-btn"></div>
                                </div>
                            </div>
                        </div>
                        <div class="sequence iconfont">
                            <span :class="iconMode" @click="changeMode"></span>
                        </div>
                        <div class="musicList iconfont">
                            <span class="icon-liebiao smallerfont" @click="listShow"></span>
                        </div>
                        <div class="favor iconfont">
                            <span @click="toggleFavorite(currentSong)" class="fav" :class="getFavoriteIcon(currentSong)"></span>
                        </div>
                    </div>
                </div>
            </div>
            </div>
            <div id="secondMini" class="mini-player" v-show="!fullScreen">
                <div class="mini-control iconfont">
                    <span @click="prev" class="smaller icon-kuaitui"></span>
                    <span @click="togglePlaying" :class="playIcon"></span>
                    <span @click="next" class="smaller icon-kuaijin"></span>
                </div>
                <div class="mini-img">
                    <img :src="currentSong.image" @click="switchScreen" alt="">
                </div>
                <div class="processLyric">
                    <div class="MsongInfo">
                      <div class="songname"><span v-text="currentSong.name"></span></div>
                        <span class="Mtime">
                            <span class="Mprogress-time" v-text="format(currentTime)"></span>
                        <span class="Mseparate">/</span>
                        <span class="Mprogress-time" v-text="format(currentSong.duration)"></span>
                        </span>
                    </div>
                    <div class="playing-lyric-wrapper">
                        <div class="playing-lyric">{{playingLyric}}</div>
                    </div>
                </div> 
               <div class="sequence iconfont hide">
                  <span :class="iconMode" @click="changeMode"></span>
                </div>
            <div class="musicList iconfont hide">
                     <span class="icon-liebiao smallerfont" @click="listShow"></span>
                    </div>

            <div class="favor iconfont">
                  <span @click="toggleFavorite(currentSong)" class="fav" :class="getFavoriteIcon(currentSong)"></span>
            </div>
            </div>
            <playlist ref="playlistTwo"></playlist>
        </div>
        <audio ref="audio" :src="currentSong.url" @play="ready" @error="error" @timeupdate="updateTime" @ended="end">您的浏览器不支持 audio 标签,请更换浏览器</audio>
    </div>
</template>
<script>
import { mapGetters, mapMutations,mapActions} from "vuex";
import "../../common/fonts/iconfont.css";
import { playMode } from "../../common/js/config";
import { upset } from "../../common/js/tool";
import Lyric from "lyric-parser";
import Playlist from "../playlist/playlist";
import {playerMixin} from '../../common/js/mixin';
const progressBtnWidth = 10;

export default { 
  mixins: [playerMixin],
  data() {
    return {
      songReady: false,
      currentTime: 0,
      currentLyric: null,
      currentLineNum: 0,
      playingLyric: ""
    };
  },
  created() {
    this.mouse = {};
  },
  computed: {
    playIcon() {
      return this.playing ? "icon-bofang" : "icon-zanting";
    },
    iconMode() {
      return this.mode == playMode.sequence
        ? "icon-liebiaoxunhuan"
        : this.mode == playMode.loop ? "icon-danquxunhuan" : "icon-suiji";
    },
    percent: {
      get() {
        return this.currentTime / this.currentSong.duration;
      },
      set(newValue) {
        return newValue / this.currentSong.duration;
      }
    },
    ...mapGetters([
      "fullScreen",
      "playlist",
      "currentSong",
      "playing",
      "currentIndex",
      "mode",
      "sequenceList"
    ])
  },
  methods: {
    switchScreen() {
      this.setFullScreen(true);
    },
    progressClick(e) {
      const rect = this.$refs.progressBar.getBoundingClientRect();
      const offsetWidth = e.pageX - rect.left;
      const barWidth = this.$refs.progressBar.clientWidth;
      const percent = offsetWidth / barWidth;
      const currentTime = this.currentSong.duration * percent;
      this.$refs.audio.currentTime = currentTime;
      if (!this.playing) {
        this.togglePlaying();
      }
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000);
      }
    },
    doDown(e) {
      let el = this.$refs.progressAgo;
      this.mouse.init = true;
      this.mouse.grabx = e.clientX;
      this.mouse.width = el.clientWidth;
      console.log("我是doDown");
      e.returnValue = false; //取消事件处理程序中调用函数的默认操作
      e.cancelBubble = true; //取消冒泡
    },
    doUp() {
      this.mouse.init = false;
      console.log("123");
      this._triggerPercent();
    },
    _triggerPercent() {
      const barWidth = this.$refs.progressBar.clientWidth;
      const percent = this.$refs.progressAgo.clientWidth / barWidth;
      console.log(percent);
      this.$refs.audio.currentTime = this.currentSong.duration * percent;
      if (!this.playing) {
        this.togglePlaying();
      }
    },
    doMove(e) {
      if (!this.mouse.init) {
        return;
      }
      //开始拖拉的位置
      console.log("我是domove");
      const offsetWidth = Math.min(
        this.$refs.progressBar.clientWidth,
        Math.max(0, this.mouse.width + e.clientX - this.mouse.grabx)
      );
      let percent = 100 * offsetWidth / this.$refs.progressBar.offsetWidth;
      this.$refs.progressAgo.style.width = `${percent}%`;
      console.log("我是domove");
      e.returnValue = false; //阻止事件默认
      e.cancelBubble = true; //阻止事件冒泡
    },
    end() {
      if (this.mode === playMode.loop) {
        this.loop();
      } else {
        this.next();
      }
    },
    loop() {
      this.$refs.audio.currentTime = 0;
      this.$refs.audio.play();
      if (this.currentLyric) {
        this.currentLyric.seek(0);
      }
    },
    next() {
      // if (!this.songReady) {
      //   return;
      // }
      if (this.playlist.length === 1) {
        this.loop();
        return;
      } else {
        let index = this.currentIndex + 1;
        if (index === this.playlist.length) {
          index = 0;
        }
        this.setCurrentIndex(index);
        if (!this.playing) {
          this.togglePlaying();
        }
      }
      this.songReady = false;
    },
    prev() {
      // if (!this.songReady) {
      //   return;
      // }
      if (this.playlist.length === 1) {
        this.loop();
        return;
      } else {
        let index = this.currentIndex - 1;
        if (index === -1) {
          index = this.playlist.length - 1;
        }
        this.setCurrentIndex(index);
        if (!this.playing) {
          this.togglePlaying();
        }
      }
      this.songReady = false;
    },
    ready() {
      this.songReady = true;
    },
    error() {
      this.songReady = true;
    },
    updateTime(e) {
      this.currentTime = e.target.currentTime;
    },
    format(interval) {
      interval = interval | 0;
      const minute = (interval / 60) | 0;
      const second = this._pad(interval % 60);
      return `${minute}:${second}`;
    },
    getLyric() {
      this.currentSong
        .getLyric()
        .then(lyric => {
          console.log('lyric');
          // document.write(lyric)
          console.log(typeof(lyric))
          console.log(lyric);
          console.log('lyric');
          // if(this.currentSong.lyric!==lyric){
          //   return ;
          // }
          this.currentLyric = new Lyric(lyric, this.handleLyric);
          if (this.playing) {
            this.currentLyric.play();
          }
        })
        .catch(() => {
          this.currentLyric = null;
          this.currentLineNum = 0;
        });
    },
    handleLyric({ lineNum, txt }) {
      this.currentLineNum = lineNum;
      this.playingLyric = txt;      
    },
    _pad(num, n = 2) {
      let len = num.toString().length;
      while (len < n) {
        num = "0" + num;
        len++;
      }
      return num;
    },
    back() {
      this.setFullScreen(false);
      this.$refs.playlistTwo.hide();
    },
    open() {
      this.setFullScreen(true);
    },
    togglePlaying() {
      // if (!this.songReady) {
      //   return;
      // }
      this.setPlayingState(!this.playing);
      if (this.currentLyric) {
        this.currentLyric.togglePlay();
      }
    },
    changeMode() {
      const mode = (this.mode + 1) % 3;
      this.setPlayMode(mode);
      let list = null;
      if (mode === playMode.random) {
        list = upset(this.sequenceList);
      } else {
        list = this.sequenceList;
      }
      this.setPlaylist(list);
    },
    resetCurrentIndex(list) {
      let index = list.findIndex(item => {
        return item.id === this.currentSong.id;
      });
      this.setCurrentIndex(index);
    },
    listShow(){
        let list=this.$refs.playlistOne;
        let listTwo=this.$refs.playlistTwo;
        if(!list.showFlag)
        {list.show();listTwo.show();}
        else{
            list.hide();
            listTwo.hide();
        }
    },
    ...mapMutations({
      setFullScreen: "SET_FULL_SCREEN",
      setPlayingState: "SET_PLAYING_STATE",
      setCurrentIndex: "SET_CURRENT_INDEX",
      setPlayMode: "SET_PLAY_MODE",
      setPlaylist: "SET_PLAYLIST"
    })
  },
  watch: {
    percent(newPercent) {
      if (newPercent >= 0 && !this.mouse.init) {
        const proPercent = newPercent * 100;
        this.$refs.progressAgo.style.width = `${proPercent}%`;
      }
    },
    currentSong(newSong, oldSong) {
      if(!newSong.id){
        return ;
      }
      // if (newSong.id === oldSong.id) {
      //   return;
      // }
      if (this.currentLyric) {
        this.currentLyric.stop();
      }
      clearTimeout(this.timer);
      this.timer=setTimeout(() => {
        this.$refs.audio.play();
        this.getLyric();
      }, 1000);
    },
    playing(newPlaying) {
      const audio = this.$refs.audio;
      this.$nextTick(() => {
        newPlaying ? audio.play() : audio.pause();
      });
    },
    playingLyric(newLyric,oldLyric){
      if(oldLyric){
        let tempDom=document.querySelector('.current');
        let distanceY=tempDom.offsetTop;
        this.$refs.lyricMove.style.top=-distanceY+'px';
      }
    }
  },
  components: {
    Playlist
  }
};
</script>
<style scoped lang="less" rel="stylesheet/less">
* {
  padding: 0;
  margin: 0;
  user-select: none;
}
h2{
  max-height: 88px;
  overflow: auto;
}
.mobileShow{
  display:none;
}
.playerPage{
   z-index:999;
}
.fixed{
  position: fixed;
    left: 0;
    right: 0;
    z-index:999;
    top: -100px;
    bottom: 0;
    background: radial-gradient(#7893a1, #15181f);
}
.player {
  width: 100%;
  text-align: center;
  .normal-player {
    position: fixed;
    left: 0;
    right: 0;
    z-index:999;
    top: 0;
    bottom: 0;
    background: radial-gradient(#7893a1, #15181f);
    text-align: center;
    h2 {
      color: #fff;
      padding: 2%;
    }
    .round {
      display: inline-block;
      margin: 2%;
      width: 20%;
      vertical-align: top;
      text-align: center;
      z-index: -1;
      img {
        max-width: 300px;
        overflow: hidden;
        clear: both;
      }
      .back {
        border: 1px solid #aaa;
        font-size: 28px;
        width: 80%;
        margin: 0 auto;
        cursor: pointer;
        color: #aaa;
        line-height: 50px;
        height: 50px;
        border-radius: 28px;
        margin-top: 10px;
      }
    }
    .detail {
      display: inline-block;
      width: 43%;
      height: 500px;
      color:#aaa;
      div {
        display: inline-block;
        margin-right: 10%;
      }
    }
    .lyric {
      width: 100%;
      height:380px;
      color: #aaa;
      transform: translateX(20px);
      overflow: hidden;
      .lyricInner{
        width: 100%;
        position: relative;
      }
      .current {
        color: #fff;
      }
    }
  }
  .mini-player {
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 180;
    width: 100%;
    height: 88px;
    background: #232a2f;
    .mini-img {
      width: 21%;
      pointer-events: none;
      vertical-align: middle;
      img {
        width: 30%;
        cursor: pointer;
        pointer-events: auto;
      }
    }
    div.mini-control {
      width: 30%;
      margin-right: -5%;
      text-align: right;
      span {
        font-size: 50px;
        margin-left: 2%;
        vertical-align: middle;
        display: inline-block;
        color: rgb(166, 206, 206);
        cursor: pointer;
        font-weight: lighter;
      }
      span.smaller {
        font-size: 36px;
        color: rgb(166, 206, 206);
        cursor: pointer;
        font-weight: 100;
      }
    }
    .sequence{
      max-width:30px;
    }
    .sequence,.musicList {
      span {
        font-size: 28px;
        color: rgb(166, 206, 206);
        cursor: pointer;
        font-weight: 100;
      }
      .smallerfont{
          font-size: 31px;
          margin-left: 12px;
      }
    }
    .process {
      width: 30%;
      margin-right: 2%;
      margin-left: -6%;
      padding: 4% 0;
      text-align: right;
      .songname {
        float: left;
        font-size: 15px;
        color: #8c979c;
      }
      .time {
        display: inline-block;
        font-family: "Avenir", "Microsoft Yahei UI";
        font-size: 15px;
        .progress-time {
          color: #9dabb1;
        }
        .separate {
          display: inline-block;
          margin: 0 1px;
          color: #adbdc4;
        }
      }
      .progress-bar {
        width: 100%;
        height: 3px;
        margin-top: 5px;
        background: rgb(150, 97, 97);
        cursor: pointer;
        .bar-inner {
          display: flex;
          width: 100%;
          height: 3px;
          .progress-ago {
            height: 3px;
            background: #fff;
          }
          .progress-btn-wrapper {
            float: right;
            margin-top: -4px;
            border-radius: 50%;
            width: 10px;
            height: 10px;
            border: 1px solid #fff;
            background: #fff;
            cursor: pointer;
          }
        }
      }
    }
    .favor{
      margin:0 1%; 
      .fav{
        font-size: 25px;
        color:red;
        font-weight: 100;
        cursor: pointer;
      }
    }
  }
}
.title-color {
  color: #aaa;
}
.processLyric {
   margin-left: -5%;
   padding-left:2%;
   height: 100%;
  overflow: hidden;
  .MsongInfo {
    float: left;
    padding-top:5px;
    color: #aaa;
    font-size: 15px;
    height:40%;
    overflow: hidden;
    .Mtime {
      font-size: 12px;
      height:50%;
      display:block;
      overflow: hidden;
    }
  }
  color: #fff;
  .playing-lyric-wrapper {
    clear: both;
    .playing-lyric{
      width: 260px;
      font-size:15px;
      height: 37px;
      text-align: left;
      overflow: hidden;
    }
  }
}
.sequence{
  span{
    margin:0 3px;
  }
}
.favor{
    display: inline-block;
    .fav{
      color:red;
      font-weight: 300;
    }
  }

@media screen and (max-width: 750px){
  #player {
    z-index:999;
  .normal-player {
    h2{
      font-size: 20px;
      font-weight: 200;
    }    
    .round {
      margin: 0;
      width: 100%;
      z-index:1;
      vertical-align: top;
      text-align: center;
      .image-round{
        width:250px;
        margin: 0 auto;
         position: relative;
          &::after{
          content:' ';
          position: absolute;
          top:31%;
          left:31%;
          width:30%;
          border-radius: 50%;
          border:8px double rgb(233, 90, 90);
          height:30%;
          background: radial-gradient(#424b50, #000000);
        }
      }
      img {
        width:100%;
        overflow: hidden;
        clear: both;
        border-radius: 50%;
        animation:rot 6s linear infinite; 
      }
      .back {
        border: 1px solid #aaa;
        font-size: 16px;
        width: 26%;
        margin: 3px auto;
        position: relative;
        z-index:9;
        color: #aaa;
        line-height: 23px;
        height: 23px;
        border-radius: 23px;
      }
    }
    .detail {
      display: inline-block;
      font-size: 15px;
      width: 90%;
      height: 500px;
      color: #aaa;
      .title-singer,.title-album {
        display: inline-block;
        margin: 0 2px;
      }
    }
     .lyric {
      position: relative;
      width: 100%;
      color: #aaa;
      overflow: hidden;
      div.lyric-wrapper{
      width: 100%;
      margin-left:-6%;;
      overflow: hidden;
      .lyricInner{
        position: relative;
        top:0px;
        font-size: 15px;
        .current {
        color: #fff;
      }
      } 
    }
      
    }
  }
  .mini-player {
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 180;
    width: 100%;
    height: 68px;
    background: #232a2f;
    div.mini-control {
      width: 37%;
      text-align: right;
      max-width: 100px;
      margin-right: -8%;
      span {
        font-size: 32px;
        margin-left: 1%;
        vertical-align: middle;
        display: inline-block;
        color: rgb(141, 92, 92);
        cursor: pointer;
        font-weight: lighter;
      }
      span.smaller {
        font-size: 23px;
        color: rgb(150, 97, 97);
        font-weight: 100;
      }
    }
    .sequence,.musicList,.favor{
      margin: 0;
      padding: 0;
      min-width: 26px;
      span {
        font-size: 22px;
        margin: 0 1%;
        color: rgb(150, 97, 97);
        font-weight: 100;
      }
       .fav{
      color:red;
    }
    }
   
    .process {
      width: 40%;
      margin-right: 2%;
      text-align: right;
      .songname {
        float: left;
        text-align:left;
        font-size: 13px;
        height:16px;
        width:100%;
        overflow: hidden;
        margin-top:-8px; 
        color:rgb(214, 147, 147);
      }
      .time {
        display: block;
        font-family: "Avenir", "Microsoft Yahei UI";
        font-size: 12px;
        border:0;
        margin:8px; 
        .progress-time {
         color:rgb(214, 147, 147);
        }
        .separate {
          display: inline-block;
          color:rgb(214, 147, 147);
        }
      }
      .progress-bar {
        width: 100%;
        height: 3px;
        margin-top: -5px;
        background: rgb(150, 97, 97);
        .bar-inner {
          display: flex;
          width: 100%;
          height: 3px;
          .progress-ago {
            height: 3px;
            background: #fff;
          }
          .progress-btn-wrapper {
            float: right;
            margin-top: -4px;
            border-radius: 50%;
            width: 10px;
            height: 10px;
            border: 1px solid #fff;
            background: #fff;
          }
        }
      }
    }
    .icon-liebiao{
      margin-right: 8px;
    }
  }
}
.title-color {
  color: #aaa;
}
.processLyric {
  margin-left: -5%;
   width:70%;
   overflow: hidden;
  .MsongInfo {
    float: left;
    color: #aaa;
    height:45%;
    overflow: hidden;
    font-size: 15px;
    .songname{
      height:45%;
    overflow: hidden;
    }
    .Mtime {
      display:block;
      font-size: 12px;
      height:45%;
      overflow: hidden;
    }
  }
  color: #fff;
   .playing-lyric-wrapper {
    clear: both;
    .playing-lyric{
      width: 150px;
      font-size:10px;
      height: 36px;
      overflow: hidden;
      text-align: left;
    }
  }
}
div#secondMini.mini-player{
  margin:0 auto;
  div.mini-control.iconfont{
  max-width: 120px;
  min-width: 110px;
  text-align: left;
}
  .mini-img {
      width: 30%;
      vertical-align: middle;
      img {
        width: 50%;
        max-width: 56px;
        min-width: 46px;
        cursor: pointer;
        pointer-events: auto;
      }
    }
    .MsongInfo{
      font-size:12px;
    }
    .sequence{
  span{
    margin:0 3px;
  }
}
.favor{
    display: inline-block;
    .fav{
      color:red;
      font-weight: 300;
      margin:0 3px;
    }
  }
}
.hide{
  display: none;
}


.PC-Show{
  display:none;
}
.mobileShow{
  display:inline-block;
}
}

@media screen and (max-width: 280px){
  .process{
    display: none;
  }
}
  @keyframes rot {
            0%{

            }
            100%{
                transform:rotateZ(360deg);
            }
        }
</style>
