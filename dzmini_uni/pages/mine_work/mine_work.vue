<template>
<view>
<!--<import src="../common_temp/audio_player.wxml"></import>-->
<view class="messageList borderShowdow">
  <block v-for="(cellItem, cellIdx) in datalist" :key="cellIdx">
    <view class="classCell" :id="cellIdx" @tap="cellClick">
      <image class="class-cell-avatar" :src="userAvatar + '' + cellItem.authorid"></image>

      <view class="class-cell-contentView">
        <view class="username">
          {{cellItem.nickname?cellItem.nickname:cellItem.author}}
        </view>
        <view class="message">{{cellItem.subject}}
          <block v-if="cellItem.displayorder == -1">
            <text class="t-status">(回收站)</text>
          </block>
          <block v-if="cellItem.displayorder == -2">
            <text class="t-status">(审核中)</text>
          </block>
        </view>

        <view class="attachementView" v-if="cellItem.attachments.length > 0">

          <block v-for="(attItem, imageidx) in cellItem.imageA" :key="imageidx">
            <image class="attachement" :src="attItem.attachurl" :id="imageidx" :data-cellindex="cellIdx" @tap.stop="lookImage" v-if="(cellItem.videoA && imageidx < 8) || (!cellItem.videoA && imageidx < 9)"></image>
          </block>
          <block v-for="(video, aidx) in cellItem.videoA" :key="aidx">
            <image @tap.stop="clickVideo" :id="video.attachurl" class="attachement" src="https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/videoPlace.png"></image>
          </block>

          <block v-if="cellItem.imageA.length == 0 && cellItem.videoA.length == 0 && cellItem.audioA">
            <block v-for="(audio, aid) in cellItem.audioA" :key="aid">
              <block data-type="template" data-is="audio_player" data-attr="...audio.toolUse">
  <view class="audio-playView">
    <image :src="is_play && currentAudio == attachment? 'https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/audio_play.png': 'https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/audio_paused.png'" class="audio-icon audio-play" :data-listindex="listIndex" @tap.stop="audio_play" :id="attachment"></image>
    <view class="control-process">
      <text class="current-process">
        <block v-if="current_process.length > 0 && currentAudio == attachment">
        {{current_process}}
        </block>
        <block v-else>
        00:00
        </block>
      </text>
      
      <text class="total-process">
        <block>
        {{total_process}}
        </block>
      </text>
      <slider class="slider" @change="hanle_slider_change" @touchstart="handle_slider_move_start" @touchend="handle_slider_move_end" min="0" block-size="12" block-color="#6199fd" :max="slider_max" activeColor="#FFD100" backgroundColor="rgb(153,153,153,0.4)" :value="(currentAudio == attachment)?slider_value:0"></slider>

    </view>
  </view>
</block>
            </block>
          </block>

        </view>
        <rich-text class="class-cell-time" :nodes="cellItem.dateline"></rich-text>
        <view class="toolRight">
          <view class="shareHidden borderShowdow" v-if="cellItem.shareshow == 1">
            <button open-type="share" plain="true" @tap.stop="listShare" :id="cellIdx">
              <image class="shareIcon" src="https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/workshare.png"></image>
            </button>
            <button plain="true" @tap.stop="disVoice" :id="cellIdx">
              <image class="shareIcon" src="https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/workvoice.png"></image>
            </button>
          </view>
          <image src class="discIcon" @tap.stop="discClick" :data-tid="cellItem.tid" :id="cellIdx"></image>
        </view>
      </view>
    </view>
  </block>
  <view class="emptyView" :hidden="!(datalist.length <= 0)">
    <view style="width:100%;height:10rpx"></view>
    <image src="https://mc.wpnicer.com/source/plugin/mobile/template/image/xiaochengxu/minimg/mineworkEmpty.png"></image>
    <view class="emptyMsg">
      真正的懒神就是一篇内容都没有
    </view>
  </view>
</view>
<view class="noMoreView" :hidden="!noMore">
  别再用力了，你已经滑到底啦
</view>
<video id="myVideo" autoplay="true" @play="playVideo" @fullscreenchange="fullScreenChange" @ :src="currentVideo">
</video>
</view>
</template>

<script>
// pages/mine_work.js
const myWorkUrl = require("../../config").myWorkUrl;
const userAvatar = require("../../config").userAvatar;
const util = require("../../utils/util.js");
const app = getApp();
var self;

export default {
  data() {
    return {
      userAvatar,
      pagenum: 1,
      member_nickname: null,
      is_reply: false,
      noMore: false,
      datalist: "",
      is_moving_slider: false,
      currentAudio: "",
      currentVideo: ""
    };
  },

  components: {},
  props: {},

  /**
   * 生命周期函数--监听页面加载
   */
  onLoad: function (options) {
    self = this;
    self.is_reply = options.is_reply ? options.is_reply : false;

    if (self.is_reply) {
      wx.setNavigationBarTitle({
        title: "我的回复"
      });
    }

    self.requestData();
    self.videoContext = wx.createVideoContext('myVideo');
    self.setupAudioPlayer();
  },
  onPullDownRefresh: function () {
    self.requestMore(false);
  },

  /**
   * 页面上拉触底事件的处理函数
   */
  onReachBottom: function () {
    self.requestMore(true);
  },
  methods: {
    requestData() {
      var type = this.is_reply ? 'reply' : 'thread';
      console.log(type);
      app.globalData.apimanager.getRequest(myWorkUrl, {
        page: self.pagenum,
        type: type
      }).then(res => {
        console.log(res);
        wx.stopPullDownRefresh();
        var threadlist = res.Variables.data ? res.Variables.data : [];

        if (!threadlist) {
          self.setData({
            noMore: true
          });
        } else {
          self.setData({
            noMore: false
          });
        }

        var dataArr = [];
        var usernicknames = res.Variables.usernicknames;

        for (let i = 0; i < threadlist.length; i++) {
          let thread = threadlist[i];
          thread.nickname = thread.author ? thread.author : thread.author === null ? "已删除" : "匿名";

          if (usernicknames) {
            if (usernicknames[postItem.authorid]) {
              thread.nickname = usernicknames[postItem.authorid];
            }
          }

          if (this.is_play) {
            // 回复
            let posts = thread.data;

            if (posts.length > 0) {
              dataArr = dataArr.concat(posts);
            }
          } else {
            // 主题
            dataArr = dataArr.concat(thread);
          }
        }

        for (let i = 0; i < dataArr.length; i++) {
          let postItem = dataArr[i]; // 附件处理

          let attachments = postItem.attachments;
          let imageA = [];
          let audioA = [];
          let videoA = [];

          for (let k in attachments) {
            let attItem = attachments[k];

            if (attItem.isimage == -1) {
              imageA.push(attItem);
            } else if (attItem.attachment.indexOf('mp3') != -1) {
              let total_process = '00:00';

              if (attItem.description) {
                total_process = util.formatTime(parseInt(attItem.description));
              }

              attItem['toolUse'] = {
                attachment: attItem.attachurl,
                listIndex: i,
                total_process: total_process
              };
              audioA.push(attItem);
            } else if (attItem.attachment.indexOf('mp4') != -1) {
              videoA.push(attItem);
            }
          }

          postItem['imageA'] = imageA;
          postItem['audioA'] = audioA;
          postItem['videoA'] = videoA;
        }

        if (self.pagenum > 1) {
          dataArr = self.datalist.concat(dataArr);
        }

        dataArr = dataArr ? dataArr : [];
        self.setData({
          datalist: dataArr
        });
      }).catch(res => {
        console.log(res);
        wx.stopPullDownRefresh();
      });
    },

    requestMore(isMore) {
      if (isMore) {
        self.pagenum += 1;
      } else {
        self.pagenum = 1;
      }

      self.requestData();
    },

    cellClick(e) {
      var item = this.datalist[e.currentTarget.id];
      this.toDetail(item);
    },

    toDetail(item) {
      var special = item.special;
      var tid = item.tid;
      var url = '../thread_detail/thread_detail?tid=';

      if (special == 1) {
        url = '../questionnaire_detail/questionnaire_detail?tid=';
      } else if (special == 3) {
        url = '../question_answer_detail/question_answer_detail?tid=';
      } else if (special == 4) {
        url = '../activity_detail/activity_detail?tid=';
      }

      url += tid;
      wx.navigateTo({
        url: url
      });
    },

    lookImage(e) {
      let cellItem = self.datalist[e.currentTarget.dataset.cellindex];
      let imageA = cellItem.imageA;
      var imageSrcArray = [];

      for (let i = 0; i < imageA.length; i++) {
        let item = imageA[i];
        imageSrcArray.push(item.attachurl);
      }

      wx.previewImage({
        current: imageSrcArray[e.currentTarget.id],
        urls: imageSrcArray
      });
    },

    /* *********************** 播放语音相关start *********** */
    setupAudioPlayer() {
      this.innerAudioContext = wx.createInnerAudioContext();
      this.innerAudioContext.obeyMuteSwitch = false;
      this.innerAudioContext.onEnded(res => {
        self.stopVoice();
      });
      this.innerAudioContext.onTimeUpdate(() => {
        if (!self.is_moving_slider) {
          // 播放中
          self.currentAudio = self.datalist[self.currentAudio.toolUse.listIndex].audioA[0];
          console.log('cur', self.currentAudio);
          let param = {};
          let toolUsestr = "datalist[" + self.currentAudio.toolUse.listIndex + "].audioA[0].toolUse";
          let toolUse = self.currentAudio.toolUse;
          toolUse['current_process'] = util.formatTime(this.innerAudioContext.currentTime);
          toolUse['slider_value'] = Math.floor(this.innerAudioContext.currentTime);
          toolUse['total_process'] = util.formatTime(this.innerAudioContext.duration);
          toolUse['slider_max'] = Math.floor(this.innerAudioContext.duration);
          param[toolUsestr] = toolUse;
          self.setData(param);
        }
      });
    },

    // 拖动进度条，到指定位置
    hanle_slider_change(e) {
      const position = e.detail.value;
      this.seekCurrentAudio(position);
    },

    // 拖动进度条控件
    seekCurrentAudio(position) {
      this.innerAudioContext.seek(position);
    },

    // 进度条滑动
    handle_slider_move_start() {
      this.setData({
        is_moving_slider: true
      });
    },

    handle_slider_move_end() {
      this.setData({
        is_moving_slider: false
      });
    },

    // 点击播放暂停
    audio_play(e) {
      let listIndex = e.currentTarget.dataset.listindex;
      let postItem = self.datalist[listIndex];
      console.log('postItem', postItem);
      var param = {};
      let audioset = "datalist[" + listIndex + "].audioA[0].toolUse.currentAudio";
      param[audioset] = postItem.audioA[0].attachurl;
      self.setData(param);
      let currentAudio = postItem.audioA[0];
      console.log('currentaudio', currentAudio);
      self.setData({
        currentAudio: currentAudio
      });
      let isplay = self.currentAudio.toolUse.is_play;
      let playstr = "datalist[" + listIndex + "].audioA[0].toolUse.is_play";
      param[playstr] = !isplay;
      self.setData(param);

      if (isplay) {
        self.innerAudioContext.pause();
        isplay = false;
      } else if (!isplay && !self.currentAudio.toolUse.is_ended) {
        self.playVoice();
      } else {
        self.innerAudioContext.play();
      }

      console.log('currentAudio', currentAudio);
    },

    playVoice() {
      var src = self.currentAudio.attachurl;
      this.innerAudioContext.src = src;
      this.innerAudioContext.play();
    },

    stopVoice() {
      var param = {};
      let listIndex = self.currentAudio.toolUse.listIndex;
      let toolUsestr = "datalist[" + listIndex + "].audioA[0].toolUse";
      let toolUse = self.currentAudio.toolUse;
      toolUse['is_play'] = false;
      toolUse['slider_value'] = 0;
      toolUse['current_process'] = util.formatTime(0);
      param[toolUsestr] = toolUse;
      self.setData(param);
      self.innerAudioContext.stop();
      let postItem = self.datalist[listIndex];
      self.currentAudio = postItem.audioA[0];
    },

    /* *********************** 语音end *********** */
    // 视频播放
    clickVideo(e) {
      self.setData({
        currentVideo: e.currentTarget.id
      });
      self.videoContext.play();
      self.videoContext.requestFullScreen({
        direction: 0
      });
    },

    playVideo() {},

    fullScreenChange(e) {
      console.log(e.detail);

      if (!e.detail.fullScreen) {
        self.videoContext.stop();
      }
    }

  }
};
</script>
<style>
@import "./mine_work.css";
</style>