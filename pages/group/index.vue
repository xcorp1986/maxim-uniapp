<template>
<view>
<snav :title="stitle">
  <view class="back" @tap.stop="backClick">
    <image class="back_kmg" src="/static/pages/image/back.png"></image>
  </view>
  <view class="delete_button" @click="deleteConversation()">
        删除会话
  </view>
</snav>
<view class="container" :style="'padding-top:' + navHeight + 'px'">
  <scroll-view class="contentcontainer" :style="'height:' + wh + 'px'" scroll-y :scroll-top="scrolltop">
    <block v-for="(message, index) in messages" :key="index">
      <message :message="message"></message>
    </block>
  </scroll-view>
  <view class="inputer">
    <image class="voice" src="/static/pages/image/voice.png" @tap="voiceHandler"></image>
    <input v-if="!showvoice" @input="inputChangeHandler" placeholder="type a message!" class="input" type="text" confirm-type="send" @confirm="sendMessageHandler" :value="inputValue"></input>
    <view v-else class="recorder" @touchstart="startRecord" @touchend="stopRecord">
      <text>{{recordTxt}}</text>
    </view>
  </view>
</view>
</view>
</template>

<script>
//index.js
//获取应用实例
import { toNumber, toLong } from "../../third/tools";
import message from "./message/index";

export default {
  data() {
    return {
      messages: [],
      gid: 0,
      scrolltop: 1000,
      inputValue: '',
      showing: false,
      stitle: '',
      wh: 0,
      navHeight: 0,
      showvoice: false,
      recordTxt: '按下录音',
      startTime: 0,
      duration: 0,
      timer: null,
      recordFile: ''
    };
  },

  components: {
    message
  },
  props: {},
  onShow: function () {
    this.setData({
      showing: false
    });
    getApp().getIM().groupManage.readGroupMessage(this.gid);
  },
  onUnload: function () {
    this.setData({
      showing: false
    });
	
	const im = getApp().getIM();
	im && im.off({
	  'onGroupMessage': this.receiveNewMessage,
	  'onMessageStatusChanged': this.onMessageStatusChanged,
	  'onSendingMessageStatusChanged': this.onSendingMessageStatusChanged,
	});
  },
  onLoad: function (options) {
    this.setData({
      navHeight: getApp().getNavHeight(),
	  showing: false
    });
    
    const {
      gid
    } = options;
    this.setData({
      gid: gid - 0
    });
    const im = getApp().getIM();
	if(!im) return;
	
    const messages = im.groupManage.getGruopMessage(gid - 0);
    this.appendMessage({
      messages
    });
    setTimeout(() => {
      this.scroll();
    }, 500);
	im.on({
	  'onGroupMessage': this.receiveNewMessage,
	  'onMessageStatusChanged': this.onMessageStatusChanged,
	  'onSendingMessageStatusChanged': this.onSendingMessageStatusChanged,
	});
    im.groupManage.readGroupMessage(this.gid);
    const allGroupMap = im.groupManage.getAllGroupDetail();
    const sgroup = allGroupMap[gid] || {};
    this.setData({
      stitle: sgroup.name || ''
    });
    const wh = wx.getSystemInfoSync().windowHeight - this.navHeight - 70;
    this.setData({
      wh
    });
  },
  methods: {
	onMessageStatusChanged: ({mid}) => {
	  console.log("Message status changed, mid: ", mid);
	},
	onSendingMessageStatusChanged: ({status, mid}) => {
	  console.log("Sending Message status changed to ", status," mid: ", mid);
	},
	  
    appendMessage: function (data) {
      const newMessages = data.messages || [];
      const isHistory = data.history;
      const oldMessages = this.messages || [];
      newMessages.forEach(meta => {
        isHistory && (meta.h = true);
        const {
          to
        } = meta;

        if (to != this.gid) {
          return; // rosterchat, 必须有一个id是 sid
        }

        if (oldMessages.length > 0) {
          const firstItem = oldMessages[0];
          const lastItem = oldMessages[oldMessages.length - 1];
          const compFirst = toLong(meta.id).comp(firstItem.id || 0);
          const compLast = toLong(meta.id).comp(lastItem.id || 0);

          if (compFirst === -1) {
            // 比第一个小
            oldMessages.unshift(meta);
          } else if (compLast === 1) {
            oldMessages.push(meta);
          } else {
            let index = -1;

            for (var i = 0; i < oldMessages.length - 2; i += 1) {
              const compCurr = toLong(meta.id).comp(oldMessages[i].id);
              const compNext = toLong(meta.id).comp(oldMessages[i + 1].id);

              if (compCurr === 1 && compNext === -1) {
                index = i;
              }
            }

            if (index > -1) {
              oldMessages.splice(index, 0, meta); // 插入到这里
            }
          }
        } else {
          // 数组为空
          oldMessages.push(meta);
        }
      }); // context.commit('setMessage', [].concat(oldMessages));
      // if (!isHistory && oldMessages.length !== state.messages.length) {
      //   state.scroll = state.scroll + 1;
      // }

      this.setData({
        messages: [].concat(oldMessages)
      });
    },

    receiveNewMessage(message) {
	  const im = getApp().getIM();
      const to = message.to;
      const pid = this.gid;

      if (pid == to) {
        if (!this.checkTyping(message)) {
          // const smessages = im.groupManage.getGruopMessage(pid - 0);
          this.appendMessage({
            messages: [message]
          });
          this.scroll();
        }
      }

      if (this.showing && im) {
        im.groupManage.readGroupMessage(this.gid);
      }
    },

    checkTyping(message) {
      const {
        ext = {}
      } = message;

      if (typeof ext.input_status !== "undefined") {
        let status = ext.input_status;

        if (status == "nothing") {// this.header.querySelector(".typing").style.display = "none";
        } else {// this.header.querySelector(".typing").style.display = "inline";
            // this.header.querySelector(".typing").innerHTML = status + "...";
          }

        return true;
      }

      return false;
    },

    scroll() {
      const scrolltop = this.scrolltop + this.messages.length * 1000;
      this.setData({
        scrolltop
      });
    },

    sendMessageHandler() {
      const content = this.inputValue;

      if (content) {
        getApp().getIM().sysManage.sendGroupMessage({
          content,
          gid: this.gid,
		  // ext: "自定义消息字段",
        });
        setTimeout(() => {
          this.setData({
            inputValue: ''
          });
        }, 400);
      }
    },

    inputChangeHandler(evt) {
      const inputValue = evt.detail && evt.detail.value || '';
      this.setData({
        inputValue
      });
    },

    backClick() {
      if( getCurrentPages().length > 1 ){
      	wx.navigateBack();  
      }else{
      	wx.switchTab({
      	  url: '/pages/contact/index'
      	});  
      }
    },

	deleteConversation() {
	  const also_delete_other_devices = true;	
	  getApp().getIM().sysManage.deleteConversation(this.gid, also_delete_other_devices);
	  this.backClick();
	},
	
    voiceHandler() {
      this.setData({
        showvoice: !this.showvoice
      });
    },

    startRecord() {
      wx.authorize({
        scope: "scope.record",
        success: function () {
          console.log("录音授权成功");
        },
        fail: function () {
          console.log("录音授权失败");
        }
      });
      var that = this;
      const recorderManager = wx.getRecorderManager();
      recorderManager.onError(res => {
        console.log("录音错误: ", res);
      });
      recorderManager.onStop(function (res) {
        that.setData({
          recordFile: res.tempFilePath,
          duration: Math.ceil((new Date().getTime() - that.startTime) / 1000)
        });
        that.preupload(that.recordFile);
        console.log("录音完成: ", that.recordFile);
      });
      const options = {
        sampleRate: 44100,
        numberOfChannels: 1,
        encodeBitRate: 192000,
        format: 'mp3'
      };
      recorderManager.start(options);
      this.setData({
        startTime: new Date().getTime(),
        recordTxt: '录音中'
      });
      const timer = setTimeout(() => {
        this.stopRecord();
      }, 30000);
      this.setData({
        timer
      });
    },

    stopRecord() {
      if (this.timer) {
        clearTimeout(this.timer);
        this.setData({
          timer: null
        });
      }

      const recorderManager = wx.getRecorderManager();
      recorderManager.stop();
      this.setData({
        recordTxt: '按下录音'
      });
    },

    preupload(path) {
      const that = this;
      getApp().getIM().sysManage.asyncGetFileUploadChatFileUrl({
        file_type: 104,
        to_id: this.gid,
        to_type: 2
      }).then(res => {
        that.uploadVoice(path, res);
      });
    },

    uploadVoice(path, param) {
      const that = this;
      const im = getApp().getIM();
      const token = im ? im.userManage.getToken() : undefined;
      wx.uploadFile({
        url: param.upload_url,
        filePath: path,
        name: "file",
        //后台要绑定的名称
        header: {
          "Content-Type": "multipart/form-data",
          'access-token': token,
          'app_id': getApp().getAppid()
        },
        //参数绑定
        formData: {
          OSSAccessKeyId: param.oss_body_param.OSSAccessKeyId,
          policy: param.oss_body_param.policy,
          key: param.oss_body_param.key,
          signature: param.oss_body_param.signature,
          callback: param.oss_body_param.callback
        },
        success: function (res) {
          that.sendVoiceMessage(param.download_url);
        },
        fail: function (ress) {
          console.log("。。录音保存失败。。");
        }
      });
    },

    sendVoiceMessage(url) {
      const im = getApp().getIM();
      const fileInfo = {
        dName: 'file',
        url,
        duration: this.duration
      };
      im.sysManage.sendGroupMessage({
        type: 'audio',
        gid: this.gid,
        content: "",
        attachment: fileInfo
      });
    }

  }
};
</script>
<style>
@import "./index.css";
</style>