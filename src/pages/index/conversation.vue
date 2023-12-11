<template>
  <view class="conversation-wrapper">
    <view class="message-container">
      <scroll-view
        scroll-y
        :refresher-triggered="state.isRefreshing"
        refresher-enabled
        :style="{ height: messageListHeight + 'px' }"
        :scroll-into-view="state.scrollIntoMessageId"
        @refresherrefresh="handleRefresh"
        @refresherpulling="handlePulling"
        @scrolltoupper="handleScrolltoupper">
        <view class="message-list">
          <view v-for="message in state.dataList" :key="message.messageId" :id="message.messageId">
            <message-row :isMe="message.userId === userStore.userWsid" :message="message" />
          </view>
        </view>
        <view id="message-bottom"></view>
      </scroll-view>
    </view>
    <view class="operation-container">
      <view class="message-input">
        <textarea
          maxlength="200"
          v-model="state.sendInput"
          :placeholder="props.sendHint" />
      </view>
      <view>
        <select-image-list ref="selectImageListRef" @imageListChange="handleImageListChange" />
      </view>
      <view class="send-button-container flex-center">
        <button class="ws-button" :disabled="disableSend" @click="handleSendMessageClick">发送</button>
      </view>
    </view>
  </view>
</template>
<script setup lang="ts">
import { nextTick, reactive, ref, defineExpose, watch, onUnmounted } from "vue"
import SelectImageList from "./select-image-list.vue"
import type { SelectType } from "./select-image-list.vue"
import { onHide, onLoad, onShow } from "@dcloudio/uni-app"
import MessageRow from "./message.vue"
import type { Message, MessageImage } from "./message.vue"

interface ConversationProps {
  beforeSendMessage: Function, // 发送消息前检查，添加是否登录等判断
  sendHint: string
}

const props = withDefaults(defineProps<ConversationProps>(), {
  beforeSendMessage: () => {
    return true
  },
  sendHint: "请输入消息"
})

interface State {
  dataList: Message[] // 对话消息列表
  isRefreshing: boolean
  scrollIntoMessageId: string // scroll-view滑动到的组件id
  currentPage: number // 消息列表的分页
  topMessageId: string // 滑动消息的最新消息id
  sendInput: string // 输入发送的消息
  attachFiles: MessageImage[] // 待发送的附件列表
  imageSelectorHeight: number // 图片选择器高度
}

// 图片选择器
const selectImageListRef = ref()
// 发送按钮，有图片或者文字时才能点击发送
const disableSend = ref(true)
// 循环消息获取器
const loopMessageInterval = ref()
// 当前手机的高度
const windowHeight = ref(0)
// 消息列表滑动的高度
const messageListHeight = ref(0)
// 对话人的消息高度
const otherInfoHeight = ref(0)
// 消息操作发送的高度：图片选择器的高度加上其他
// 图片选择器高度
const imageSelectorHeight = ref(90)
// 消息发送默认高度
const sendInputHeight = ref(90)
// 发送按钮的高度
const sendButtonHeight = ref(120)
// 是否上传图片，否则直接显示本地图片，是则上传
const isUploadImage = ref(false)
// 图片是否再上传
const isImageUploading = ref(false)
// 上传图片地址
const uploadImageUrl = ref("/system/upload-file")
// 登录的用户信息
const userStore = reactive({
  sessionWsid: "",
  userWsid: ""
})

const state = reactive<State>({
  dataList: [], // 消息数据
  isRefreshing: false,
  scrollIntoMessageId: "", // 消息滑动位置id
  currentPage: 1, // 分页
  topMessageId: "", // 记录要滚动消息的组件id
  sendInput: "",
  attachFiles: [],
  imageSelectorHeight: 0
})

onLoad(() => {
  const systemInfo = uni.getSystemInfoSync()
  windowHeight.value = systemInfo.windowHeight
  messageListHeight.value =
    windowHeight.value -
    otherInfoHeight.value -
    sendInputHeight.value -
    imageSelectorHeight.value -
    sendButtonHeight.value
})

watch(
  state,
  (newValue, oldValue) => {
    console.log("message input change", newValue)
    disableSend.value = !state.attachFiles.length && !state.sendInput.length
  },
  {
    immediate: true
  }
)

// 轮询刷新消息
const loopRefreshMessageList = () => {
  if (!loopMessageInterval.value) {
    loopMessageInterval.value = setInterval(() => {
      if (userStore.sessionWsid) {
        // TODO 刷新消息
      } else {
        // TODO
      }
    }, 5000)
  }
}

const removeRefreshInterval = () => {
  if (loopMessageInterval.value) {
    clearInterval(loopMessageInterval.value)
    loopMessageInterval.value = null
  }
}

const initMessageList = () => {
  state.currentPage = 1
  getMessageList(state.currentPage)
}

const getMessageList = (page: number) => {
  setTimeout(() => {
    let newDataList = new Array<Message>()
    for (let i = 1; i < 10; i++) {
      let scrollId = "s_" + page + "_" + i
      if (i % 2 === 0) {
        let imgList: MessageImage[] = []
        newDataList.push({
          userId: userStore.userWsid,
          content: "测试消息" + scrollId,
          sendTime: "",
          attachFiles: imgList,
          messageId: scrollId
        })
      } else {
        newDataList.push({
          userId: "doctor_id",
          content: "测试消息" + scrollId,
          sendTime: "",
          attachFiles: [],
          messageId: scrollId
        })
      }
    }
    // 当获取到下一个分页的聊天数据，将他第一条的id作为要移动过去的id，这样刷新之后会直接上到更新数据的第一条
    if (page > 1 && newDataList.length) {
      state.topMessageId = newDataList[newDataList.length - 1].messageId
    }
    state.dataList = newDataList.concat(state.dataList)
    uni.stopPullDownRefresh()
    state.isRefreshing = false

    scrollToMessage()
  }, 1000)
}

// 下拉刷新
const handleRefresh = () => {
  state.currentPage++
  getMessageList(state.currentPage)
}

// 下拉
const handlePulling = e => {
  if (e.detail.dy <= 0) return
  if (state.isRefreshing) return
  state.isRefreshing = true
}

// 处理：页面上任意位置只要下滑动页面就会触发下拉
const handleScrolltoupper = () => {
  state.isRefreshing = false
}

// 发送消息，如果登录失效则会登录之后再返回发送消息
const handleSendMessageClick = () => {
  if (checkSendMessageContent() && props.beforeSendMessage()) {
    sendMessageImpl()
  }
}

// 检查发送消息是否符合
const checkSendMessageContent = () => {
  if (!state.sendInput && !state.attachFiles.length) {
    uni.showToast({ title: "请输入消息或者添加图片" })
    return false
  }
  if (!state.attachFiles.length) {
    if (state.sendInput.length < 5) {
      uni.showToast({ title: "最少输入5个字" })
      return false
    }
    if (state.sendInput.length > 200) {
      uni.showToast({ title: "最多输入200字" })
      return false
    }
  }

  return true
}

/**
 * 上传单个图片
 * @param messageImage 本地的图片对象
 * @param callback 上传成功
 */
const uploadImage = (messageImage: MessageImage, callback: Function) => {
  console.log("upload messageImage", messageImage)
  messageImage.loading = true
  // TODO 上传图片
}

const sendMessageWithImage = (message: Message) => {
  let uploadedImages: string[] = []
  // 上传图片，按顺序要显示
  for (let i = 0; i < message.attachFiles.length; i++) {
    let image = message.attachFiles[i]
    console.log("upload image", image)
    uploadImage(image, messageImage => {
      if (messageImage) {
        // 上传成功，按顺序插入图片的地址
        uploadedImages.splice(i, 0, messageImage.url)
      } else {
        // 图片上传失败，添加error，让消息显示的对应图片添加遮罩显示上传失败，然后可以点击重新上传
        uploadedImages.splice(i, 0, "error")
      }
    })
  }

  resetMessageInput()
}

const sendMessageWithoutImage = (message: Message) => {
  resetMessageInput()
}

/**
 * 重置消息发区域
 */
const resetMessageInput = () => {
  state.sendInput = ""
  state.attachFiles = []
  selectImageListRef.value.resetImageList()
}

/**
 * 发送消息
 */
const sendMessageImpl = () => {
  // 消息时间戳
  let timestamp = "m_" + new Date().getTime().toString()
  // 不影响本地的消息刷新拷贝一个新的消息对象到更新的消息队列
  let message = {
    userId: userStore.userWsid,
    content: state.sendInput,
    sendTime: "",
    attachFiles: state.attachFiles,
    messageId: timestamp
  }
  let copyMessage = Object.assign({}, message)
  // 更新本地消息列表
  state.topMessageId = copyMessage.messageId
  state.dataList.push(copyMessage)
  nextTick(() => {
    scrollToMessage()
  })
  console.log("send message", copyMessage)
  // 区分带有附件的消息
  if (copyMessage.attachFiles && copyMessage.attachFiles.length) {
    sendMessageWithImage(copyMessage)
  } else {
    sendMessageWithoutImage(copyMessage)
  }
}

// 滚动消息，如果是第一次更新则滑动到最底部，如果是刷新历史消息则上滑一条消息
const scrollToMessage = () => {
  nextTick(() => {
    if (state.currentPage === 1 && state.dataList && state.dataList.length > 0) {
      // 第一页的消息
      state.scrollIntoMessageId = state.dataList[state.dataList.length - 1].messageId
    } else {
      state.scrollIntoMessageId = state.topMessageId
    }
  })
}

// 图片选择器更改，需要获取高度然后调整图片容器的高度和消息的高度
const handleImageListChange = (imageList: SelectType[]) => {
  state.attachFiles = []
  imageList.forEach(localImage => {
    let newMessageImage: MessageImage = {
      path: localImage.imgPath,
      loading: false,
      url: "",
      type: localImage.type
    }
    state.attachFiles.push(newMessageImage)
  })
  // 获取图片选择器的高度，然后缩放消息的高度
  nextTick(() => {
    const selectorHeight = selectImageListRef.value.$el.offsetHeight
    messageListHeight.value =
      windowHeight.value - otherInfoHeight.value - sendInputHeight.value - selectorHeight - sendButtonHeight.value
  })
}

onShow(() => {
  loopRefreshMessageList()
})

onHide(() => {
  removeRefreshInterval()
})

onUnmounted(() => {
  removeRefreshInterval()
})

defineExpose({ initMessageList })
</script>
<style lang="scss" scoped>
page {
  background: #fff;
}

.conversation-wrapper {
}

.message-container {
  padding: 30rpx 0;
}

.message-list {
  padding: 0 30rpx;
}

.operation-container {
  border-top: 1rpx solid #bbbbbb;
}

.send-button-container {
  margin: 0 30rpx;
  height: 120rpx;
}

.ws-button {
  width: 100%;
  height: 88rpx;
  padding: 0;
  margin: 0 auto;
  font-size: 34rpx;
  text-align: center;
  border-radius: 8rpx;
  line-height: 88rpx;
  font-weight: 500;
  letter-spacing: 4rpx;
  color: #fff !important;
  background: #0c7ffc;
  border: 2rpx solid #0c7ffc;

  &[disabled] {
    color: #fff !important;
    background: #9eccfe !important;
    border: 2rpx solid #9eccfe !important;
  }
}

.message-input {
  padding: 30rpx;

  textarea {
    width: 100%;
    height: 120rpx;
  }
}
</style>
