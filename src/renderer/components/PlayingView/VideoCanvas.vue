<template>
  <div class="video">
    <video ref="videoCanvas"
      preload="metadata"
      @playing="onPlaying"
      @pause="onPause"
      @canplay="onCanPlay"
      @timeupdate="onTimeupdate"
      @loadedmetadata="onMetaLoaded"
      @durationchange="onDurationChange"
      :src="src">
    </video>
    <div class="subtitle-wrapper">
      <!-- Need a way not to use v-html -->
      <div class='subtitle-content'
        :style="subStyle"
        v-for="(html, key) in firstCueHTML"
        v-html="html"></div>
      <div class='subtitle-content'
        :style="subStyle"
        v-for="(html, key) in secondCueHTML"
        v-html="html"></div>
    </div>
    <canvas class="canvas" ref="thumbnailCanvas"></canvas>
  </div>
</template>;

<script>
import fs from 'fs';
import srt2vtt from 'srt-to-vtt';
import { WebVTT } from 'vtt.js';
import path from 'path';
// https://www.w3schools.com/tags/ref_av_dom.asp
import parallel from 'run-parallel';
import syncStorage from '@/helpers/syncStorage';

export default {
  data() {
    return {
      windowRectangleOld: {},
      videoExisted: false,
      shownTextTrack: false,
      newWidthOfWindow: 0,
      newHeightOfWindow: 0,
      videoWidth: 0,
      videoHeight: 0,
      timeUpdateIntervalID: null,
      firstActiveCue: null,
      secondActiveCue: null,
      firstCueHTML: [],
      secondCueHTML: [],
      subNameArr: [],
      // 将style的内容修改为object
      subStyle: {},
      curStyle: {
        fontSize: 24,
        letterSpacing: 1,
        opcacity: 1,
        color: '',
        border: '',
        background: '',
      },
    };
  },
  props: {
    src: {
      type: String,
      required: true,
      validator(value) {
        // TODO: check if its a file or url
        if (value.length <= 0) {
          return false;
        }
        return true;
      },
    },
  },
  methods: {
    accurateTimeUpdate() {
      const { currentTime, duration } = this.$refs.videoCanvas;
      if (currentTime >= duration || this.$refs.videoCanvas.paused) {
        clearInterval(this.timeUpdateIntervalID);
      } else {
        this.$store.commit('AccurateTime', currentTime);
      }
    },
    onPause() {
      console.log('onpause');
    },
    onPlaying() {
      console.log('onplaying');
      // set interval to get update time
      const { duration } = this.$refs.videoCanvas;
      if (duration <= 240) {
        this.timeUpdateIntervalID = setInterval(this.accurateTimeUpdate, 10);
      }
    },
    onCanPlay() {
      // the video is ready to start playing
      this.$store.commit('Volume', this.$refs.videoCanvas.volume);
    },
    onMetaLoaded() {
      console.log('loadedmetadata');
      this.$bus.$emit('play');
      this.$bus.$emit('seek', this.currentTime);
      this.videoWidth = this.$refs.videoCanvas.videoWidth;
      this.videoHeight = this.$refs.videoCanvas.videoHeight;
      this.$bus.$emit('screenshot-sizeset', this.videoWidth / this.videoHeight);
      if (this.videoExisted) {
        this.$_calculateWindowSizeInConditionOfVideoExisted();
        this.$_controlWindowSizeAtNewVideo();
      } else {
        this.$_calculateWindowSizeAtTheFirstTime();
        this.$_controlWindowSize();
        this.videoExisted = true;
      }

      this.loadTextTracks();
    },
    onTimeupdate() {
      console.log('ontimeupdate');
      this.$store.commit('AccurateTime', this.$refs.videoCanvas.currentTime);
      const t = Math.floor(this.$refs.videoCanvas.currentTime);
      if (t !== this.$store.state.PlaybackState.CurrentTime) {
        this.$store.commit('CurrentTime', t);
      }
    },
    onDurationChange() {
      console.log('durationchange');
      const t = Math.floor(this.$refs.videoCanvas.duration);
      if (t !== this.$store.state.PlaybackState.duration) {
        this.$store.commit('Duration', t);
      }
    },
    onSubTrackLoaded() {
      console.log('onSubTrackLoaded');
      this.$refs.customTrack.mode = 'showing';
    },
    $_controlWindowSize() {
      const currentWindow = this.$electron.remote.getCurrentWindow();
      currentWindow.setBounds({
        x: 0,
        y: 0,
        width: parseInt(this.newWidthOfWindow, 10),
        height: parseInt(this.newHeightOfWindow, 10),
      });
      currentWindow.setAspectRatio(this.newWidthOfWindow / this.newHeightOfWindow);
    },
    $_controlWindowSizeAtNewVideo() {
      const currentWindow = this.$electron.remote.getCurrentWindow();
      const windowXY = this.calcNewWindowXY();
      currentWindow.setBounds({
        x: windowXY.windowX,
        y: windowXY.windowY,
        width: parseInt(this.newWidthOfWindow, 10),
        height: parseInt(this.newHeightOfWindow, 10),
      });
      currentWindow.setAspectRatio(this.newWidthOfWindow / this.newHeightOfWindow);
    },
    $_calculateWindowSizeAtTheFirstTime() {
      const currentWindow = this.$electron.remote.getCurrentWindow();
      const currentScreen = this.$electron.screen.getPrimaryDisplay();

      const { width: screenWidth, height: screenHeight } = currentScreen.workAreaSize;
      const [minWidth, minHeight] = currentWindow.getMinimumSize();
      const screenRatio = screenWidth / screenHeight;
      const minWindowRatio = minWidth / minHeight;
      const videoRatio = this.videoWidth / this.videoHeight;

      if (this.videoWidth > screenWidth || this.videoHeight > screenHeight) {
        if (videoRatio > screenRatio) {
          this.newWidthOfWindow = screenWidth;
          this.newHeightOfWindow = this.calculateHeightByWidth;
        } else if (videoRatio < screenRatio) {
          this.newHeightOfWindow = screenHeight;
          this.newWidthOfWindow = this.calculateWidthByHeight;
        } else if (videoRatio === screenRatio) {
          [this.newWidthOfWindow, this.newHeightOfWindow] = [screenWidth, screenHeight];
        }
      } else if (this.videoWidth < minWidth || this.videoHeight < minHeight) {
        if (videoRatio > minWindowRatio) {
          this.newHeightOfWindow = minHeight;
          this.newWidthOfWindow = this.calculateWidthByHeight;
        } else if (videoRatio < minWindowRatio) {
          this.newWidthOfWindow = minWidth;
          this.newHeightOfWindow = this.calculateHeightByWidth;
        } else if (videoRatio === minWindowRatio) {
          [this.newWidthOfWindow, this.newHeightOfWindow]
            = [this.videoWidth, this.videoHeight];
        }
      } else {
        [this.newWidthOfWindow, this.newHeightOfWindow] = [this.videoWidth, this.videoHeight];
      }
    },

    $_calculateWindowSizeInConditionOfVideoExisted() {
      const currentWindow = this.$electron.remote.getCurrentWindow();

      const [windowWidth, windowHeight] = currentWindow.getSize();
      const [minWidth, minHeight] = currentWindow.getMinimumSize();

      const windowRatio = windowWidth / windowHeight;
      const minWindowRatio = minWidth / minHeight;
      const videoRatio = this.videoWidth / this.videoHeight;

      if (this.videoWidth < windowWidth && this.videoHeight < windowHeight) {
        [this.newWidthOfWindow, this.newHeightOfWindow] = [this.videoWidth, this.videoHeight];
      } else if (this.videoWidth > windowWidth || this.videoHeight > windowHeight) {
        if (videoRatio > windowRatio) {
          this.newWidthOfWindow = windowWidth;
          this.newHeightOfWindow = this.calculateHeightByWidth;
        } else if (videoRatio < windowRatio) {
          this.newHeightOfWindow = windowHeight;
          this.newWidthOfWindow = this.calculateWidthByHeight;
        } else if (videoRatio === windowRatio) {
          [this.newWidthOfWindow, this.newHeightOfWindow]
            = [windowWidth, windowHeight];
        }
      }

      if (this.newWidthOfWindow < minWidth || this.newHeightOfWindow < minHeight) {
        if (videoRatio > minWindowRatio) {
          this.newHeightOfWindow = minHeight;
          this.newWidthOfWindow = this.calculateWidthByHeight;
        } else if (videoRatio < minWindowRatio) {
          this.newWidthOfWindow = minWidth;
          this.newHeightOfWindow = this.calculateHeightByWidth;
        } else if (videoRatio === minWindowRatio) {
          [this.newWidthOfWindow, this.newHeightOfWindow]
            = [this.videoWidth, this.videoHeight];
        }
      }
      console.log(this.newWidthOfWindow);
    },
    $_saveScreenshot() {
      const canvas = this.$refs.thumbnailCanvas;
      const canvasCTX = canvas.getContext('2d');
      const { videoHeight, videoWidth } = this.$refs.videoCanvas;
      [canvas.width, canvas.height] = [videoWidth, videoHeight];

      canvasCTX.drawImage(
        this.$refs.videoCanvas, 0, 0, videoWidth, videoHeight,
        0, 0, videoWidth, videoHeight,
      );
      const imagePath = canvas.toDataURL('image/png');
      let data;
      try {
        data = syncStorage.getSync('recent-played');
      } catch (err) {
        console.error(err);
      }
      const object = data[0];
      const iterator = Object.keys(object).indexOf('path');
      if (iterator !== -1) {
        object.shortCut = imagePath;
        object.lastPlayedTime = this.currentTime;
        object.duration = this.$store.state.PlaybackState.Duration;
        data.splice(0, 1);
        data.unshift(object);
      }
      syncStorage.setSync('recent-played', data);
    },
    /**
     * @param callback Has two parameters, err and result
     * It is function that runs after all buffers are concated.
     */
    concatStream(stream, callback) {
      const chunks = [];
      stream.on('data', (chunk) => {
        chunks.push(chunk);
      });
      stream.on('end', () => {
        if (callback) {
          callback(null, Buffer.concat(chunks));
        }
        callback = null;
      });
      stream.once('error', (err) => {
        if (callback) {
          callback(err);
        }
        callback = null;
      });
    },
    $_subNameProcess(file, index) {
      return {
        title: path.parse(file).name,
        index,
      };
    },
    /**
     * Todo:
     * 1. process ass subtitles
     * 2. second subtitle css
     * 3. accept subtitle from drag event and menu
     * 4. detect subtitle language
     */
    /**
     * Load all available text tracks in the
     * same path
     */
    loadTextTracks() {
      /* TODO:
       * 字幕代码我自己觉得很不满意，期待更好的处理 - Tomasen
       * move subtitle process to another component
       * DOCs:
       * https://gist.github.com/denilsonsa/aeb06c662cf98e29c379
       * https://developer.mozilla.org/en-US/docs/Web/API/VTTCue
       * https://hacks.mozilla.org/2014/07/adding-captions-and-subtitles-to-html5-video/
       */

      const vid = this.$refs.videoCanvas;
      const startIndex = vid.textTracks.length;
      this.$store.commit('StartIndex', startIndex);
      // hide every text text/subtitle tracks at beginning
      // 没有做对内挂字幕的处理
      for (let i = this.$store.state.PlaybackState.CurrentIndex; i < startIndex; i += 1) {
        vid.textTracks[i].mode = 'disabled';
      }

      // // create our own text/subtitle track
      // const sub0 = vid.addTextTrack('subtitles', 'splayer-custom');

      /*
       * TODO:
       * If there is already text track, load it
       * If there is already subtitle files(opened or loaded), load it
       * If there is no (chinese/default language) text track, try translate api
       */

      // If there is already subtitle files(same dir), load it

      this.$_clearSubtitle();
      console.log('heyyyyyyy');
      const files = [];
      this.findSubtitleFilesByVidPath(decodeURI(vid.src), (subPath) => {
        files.push(subPath);
      });


      const subNameArr = files.map((file, index) => this.$_subNameProcess(file, index));
      this.$store.commit('SubtitleNameArr', subNameArr);
      this.$bus.$emit('subName-loaded');

      /**
       * Referenced from WebTorrent
       */
      /* eslint-disable arrow-parens */
      const tasks = files.map((subPath) => (cb) => this.subPathProcess(subPath, cb));
      parallel(tasks, (err, results) => {
        if (err) {
          console.error(err);
        }
        const parser = new WebVTT.Parser(window, WebVTT.StringDecoder());
        for (let i = 0; i < results.length; i += 1) {
          const sub = vid.addTextTrack('subtitles');
          sub.mode = 'disabled';
          parser.oncue = (cue) => {
            sub.addCue(cue);
          };
          parser.onflush = () => {
            console.log('finished reading subtitle files');
            this.subStyleChange();
            this.subtitleShow(startIndex);
          };
          const result = results[i];
          parser.parse(result.toString('utf8'));
        }
        parser.flush();
      });
    },
    subPathProcess(subPath, cb) {
      const vttStream = fs.createReadStream(subPath).pipe(srt2vtt());
      this.concatStream(vttStream, (err, buf) => {
        if (err) {
          console.error(err);
        }
        cb(null, buf);
      });
    },
    // 待改善函数结构
    // 判断重合情况
    subtitleShow(index, type = 'first') {
      const vid = this.$refs.videoCanvas;
      if (type === 'first') {
        // 当直接播放无字幕视频时，会报错,需要error handle
        // 判断有无字幕
        if (vid.textTracks.length > index) {
          this.$store.commit('FirstSubtitleOn');
          const firstSubIndex = this.$store.state.PlaybackState.FirstSubIndex;
          vid.textTracks[firstSubIndex].mode = 'disabled';
          vid.textTracks[firstSubIndex].oncuechange = null;
          this.$_onCueChangeEventAdd(vid.textTracks[index]);
          vid.textTracks[index].mode = 'hidden';
          this.$store.commit('FirstSubIndex', index);
        } else {
          this.$store.commit('FirstSubtitleOff');
          console.log('no subtitle');
        }
      }
      if (type === 'second') {
        // 需要重写
        console.log('showSecondSub');
        const secondSubIndex = this.$store.state.PlaybackState.SecondSubIndex;
        if (index !== this.$store.state.PlaybackState.FirstSubIndex) {
          if (secondSubIndex !== -1) {
            vid.textTracks[secondSubIndex].mode = 'disabled';
            this.$store.commit('SecondSubtitleOn');
          } else {
            this.$store.commit('SecondSubtitleOff');
            console.log('Warning: no selected second subtitle');
          }
          this.$_onCueChangeEventAdd(vid.textTracks[index], 'second');
          vid.textTracks[index].mode = 'hidden';
          this.$store.commit('SecondSubIndex', index);
        }
      }
    },
    /**
     * @param obj contains all needed style property
     *
     * obj contains 6 values to control the css
     * style of the subtitle. Each unset property
     * will use default value.
     */
    subStyleChange(obj = {}) {
      const fontSize = obj.fontSize ? obj.fontSize : this.curStyle.fontSize;
      const letterSpacing = obj.letterSpacing ? obj.letterSpacing : this.curStyle.letterSpacing;
      const opacity = obj.opacity ? obj.opacity : this.curStyle.opacity;
      const color = obj.color ? obj.color : this.curStyle.color;
      const border = obj.border ? obj.border : this.curStyle.border;
      const background = obj.background ? obj.background : this.curStyle.background;

      this.subStyle = {
        fontSize: `${fontSize}px`,
        letterSpacing: `${letterSpacing}px`,
        opacity,
        color,
        border,
        background,
      };
      this.curStyle = {
        fontSize,
        letterSpacing,
        opacity,
        color,
        border,
        background,
      };
    },
    /**
     * @param textTrack target textTrack
     * @param type choose first or second subtitle
     */
    $_onCueChangeEventAdd(textTrack, type = 'first') {
      const firstSubEvent = (cue) => {
        const tempCue = cue.currentTarget.activeCues[0];
        this.firstActiveCue = tempCue;
      };
      const secondSubEvent = (cue) => {
        const tempCue = cue.currentTarget.activeCues[0];
        this.secondActiveCue = tempCue;
      };
      // write a same sub event to handle same subtitle situation
      textTrack.oncuechange = type === 'first' ? firstSubEvent : secondSubEvent;
    },
    /**
     * need to process subtitle init event
     */
    $_clearSubtitle() {
      const vid = this.$refs.videoCanvas;
      const firstSubIndex = this.$store.state.PlaybackState.FirstSubIndex;
      const secondSubIndex = this.$store.state.PlaybackState.SecondSubIndex;
      if (vid.textTracks[firstSubIndex]) {
        vid.textTracks[firstSubIndex].mode = 'disabled';
        vid.textTracks[firstSubIndex].oncuechange = null;
      }
      // 未设置第二字幕时，消除字幕的error handler
      if (secondSubIndex === -1) {
        console.log('second subtitle not set');
      } else {
        vid.textTracks[secondSubIndex].mode = 'disabled';
        vid.textTracks[secondSubIndex].oncuechange = null;
        this.$store.commit('SecondSubIndex', -1);
      }
    },
    calcNewWindowXY() {
      if (Object.keys(this.windowRectangleOld).length === 0) {
        return { windowX: 0, windowY: 0 };
      }
      let x = this.windowRectangleOld.x + (this.windowRectangleOld.width / 2);
      let y = this.windowRectangleOld.y + (this.windowRectangleOld.height / 2);
      x = Math.round(x - (this.newWidthOfWindow / 2));
      y = Math.round(y - (this.newHeightOfWindow / 2));
      return { windowX: x, windowY: y };
    },
  },
  computed: {
    calculateHeightByWidth() {
      return this.newWidthOfWindow / (this.videoWidth / this.videoHeight);
    },
    calculateWidthByHeight() {
      return this.newHeightOfWindow * (this.videoWidth / this.videoHeight);
    },
    currentTime() {
      return this.$store.state.PlaybackState.CurrentTime;
    },
    firstSubState() {
      return this.$store.state.PlaybackState.FirstSubtitleState;
    },
    secondSubState() {
      return this.$store.state.PlaybackState.SecondSubtitleState;
    },
  },
  watch: {
    firstSubState(newVal) {
      const vid = this.$refs.videoCanvas;
      const firstSubIndex = this.$store.state.PlaybackState.FirstSubIndex;
      if (newVal && vid.textTracks[firstSubIndex].mode === 'disabled') {
        vid.textTracks[firstSubIndex].mode = 'hidden';
      } else if (!newVal && vid.textTracks[firstSubIndex].mode !== 'disabled') {
        this.firstActiveCue = null;
        vid.textTracks[firstSubIndex].mode = 'disabled';
      } else {
        console.log('Error: mode is not correct');
      }
    },
    secondSubState(newVal) {
      const vid = this.$refs.videoCanvas;
      const secondSubIndex = this.$store.state.PlaybackState.SecondSubIndex;
      // 如果未选第二字幕的时候开启，如何处理
      if (secondSubIndex === -1) {
        console.log('Warn: No Second Subtitle');
      } else if (newVal && vid.textTracks[secondSubIndex].mode === 'disabled') {
        vid.textTracks[secondSubIndex].mode = 'hidden';
      } else if (!newVal && vid.textTracks[secondSubIndex].mode !== 'disabled') {
        this.secondActiveCue = null;
        vid.textTracks[secondSubIndex].mode = 'disabled';
      } else {
        console.log('Error: mode is not correct');
      }
    },
    // 需要对这一部分内容优化
    firstActiveCue(newVal) {
      this.firstCueHTML.pop();
      // console.log(newVal);
      if (newVal) {
        // 这里对cue进行处理
        // 得到cue的line和position确定位置
        this.firstCueHTML.push(WebVTT.convertCueToDOMTree(window, this.firstActiveCue.text)
          .innerHTML);
      }
    },
    secondActiveCue(newVal) {
      this.secondCueHTML.pop();
      // console.log(newVal);
      if (newVal) {
        // 这里对cue进行处理
        // 得到cue的line和position确定位置
        this.secondCueHTML.push(WebVTT.convertCueToDOMTree(window, this.secondActiveCue.text)
          .innerHTML);
      }
    },
    src() {
      const window = this.$electron.remote.getCurrentWindow();
      this.windowRectangleOld.x = window.getBounds().x;
      this.windowRectangleOld.y = window.getBounds().y;
      this.windowRectangleOld.height = window.getBounds().height;
      this.windowRectangleOld.width = window.getBounds().width;
    },
  },
  created() {
    this.$bus.$on('playback-rate', (newRate) => {
      console.log(`set video playbackRate ${newRate}`);
      this.$refs.videoCanvas.playbackRate = newRate;
      this.$store.commit('PlaybackRate', newRate);
    });
    this.$bus.$on('volume', (newVolume) => {
      console.log(`set video volume ${newVolume}`);
      this.$refs.videoCanvas.volume = newVolume;
      this.$store.commit('Volume', newVolume);
    });
    this.$bus.$on('reset-windowsize', () => {
      this.$_controlWindowSize(this.newWidthOfWindow, this.newHeightOfWindow);
    });
    this.$bus.$on('toggle-playback', () => {
      console.log('toggle-playback event has been triggered');
      if (this.$refs.videoCanvas.paused) {
        this.$bus.$emit('play');
        this.$bus.$emit('twinkle-play-icon');
      } else {
        this.$bus.$emit('pause');
        this.$bus.$emit('twinkle-pause-icon');
      }
    });
    this.$bus.$on('play', () => {
      console.log('play event has been triggered');
      this.$refs.videoCanvas.play();
    });
    this.$bus.$on('pause', () => {
      console.log('pause event has been triggered');
      this.$refs.videoCanvas.pause();
    });
    this.$bus.$on('seek', (e) => {
      console.log('seek event has been triggered', e);
      this.$refs.videoCanvas.currentTime = e;
      this.$store.commit('CurrentTime', e);
      this.$store.commit('AccurateTime', e);
    });
    // 可以二合一
    this.$bus.$on('subFirstChange', (targetIndex) => {
      const index = this.$store.state.PlaybackState.StartIndex + targetIndex;
      this.subtitleShow(index, 'first');
    });
    this.$bus.$on('subSecondChange', (targetIndex) => {
      const index = this.$store.state.PlaybackState.StartIndex + targetIndex;
      // 增加一个判断第一字幕是否开启的状态
      this.subtitleShow(index, 'second');
    });

    this.$bus.$on('subStyleChange', this.subStyleChange);
  },
  mounted() {
    window.onbeforeunload = () => {
      this.$_saveScreenshot();
    };
  },
};
</script>

<style lang="scss">

.video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 4px;
  overflow: hidden;

  video {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
}

.video {
  .subtitle-wrapper {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;

    .subtitle-content {
      position: absolute;
      bottom: 20px;
      text-align: center;
      width: 100%;
      white-space: pre;
    }
  }
}
</style>
