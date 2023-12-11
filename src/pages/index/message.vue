<template>
  <view v-if="props.isMe" class="message-line">
    <!--自己的消息-->
    <view class="message-line-self">
      <view class="message-content-container self">
        <view class="message-content-text self" v-if="props.message.content && props.message.content.length">
          <view>{{ props.message.content }}</view>
        </view>
        <view
          class="message-content-files self"
          :class="props.message.content ? 'mt-12' : 'mt-0'"
          v-if="props.message.attachFiles && props.message.attachFiles.length">
          <view
            v-for="(image, index) in props.message.attachFiles"
            :key="index"
            class="image-container"
            @click="handleImageClick(index)">
            <ws-blob-image
              :emit-path="true"
              :default-src="image.path"
              :src="image.url"
              :index="index"
              @downloaded="handleImageDownloaded" />
            <view class="spin-container" v-if="image.loading">
              <div class="spin"></div>
            </view>
          </view>
        </view>
      </view>
    </view>
    <view class="message-avatar">
      <image src="@/static/ic_doctor.png" />
    </view>
  </view>
  <view v-else class="message-line">
    <view class="message-avatar">
      <image src="@/static/ic_doctor.png" />
    </view>
    <!--对方信息-->
    <view class="message-line-other">
      <view class="message-content-container other">
        <view class="message-content-text other" v-if="props.message.content && props.message.content.length">
          <view>{{ props.message.content }}</view>
        </view>
        <view
          class="message-content-files other"
          :class="props.message.content ? 'mt-12' : 'mt-0'"
          v-if="props.message.attachFiles && props.message.attachFiles.length">
          <view
            v-for="(image, index) in props.message.attachFiles"
            :key="index"
            class="image-container"
            @click="handleImageClick(index)">
            <image :src="image.url" />
            <view class="spin-container" v-if="image.loading">
              <div class="spin"></div>
            </view>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>
<script setup lang="ts">
import { ref } from "vue"
import { AttachTypeEnum } from "./config"

// 消息中的图片对象
export interface MessageImage {
  path: string // 本地路径
  loading: boolean // 是否再上传
  url: string // 上传的地址url
  type: AttachTypeEnum // 上传的文件类型
}

// 对话信息
export interface Message {
  userId: string // 讲话人的id
  content: string
  sendTime: string
  attachFiles: MessageImage[]
  messageId: string
}

interface SelectImageListProps {
  message: Message
  isMe: boolean
}

const props = withDefaults(defineProps<SelectImageListProps>(), {
  message: () => {
    return {
      userId: "",
      content: "",
      sendTime: "",
      attachFiles: [],
      messageId: ""
    }
  },
  isMe: false
})

// 预览的图片列表
const previewImageList = ref<string[]>([])

/**
 * 预览图片
 * @param imageList 消息中图片列表
 * @param index 当前打开的位置
 */
const handleImageClick = (index: number) => {
  uni.previewImage({
    urls: previewImageList.value,
    current: index
  })
}

/**
 * 消息的图片组件下载完成消息
 * @image 下载的本地地址
 * @index 当前图片下载之前的位置
 */
const handleImageDownloaded = (image: string, index: number) => {
  previewImageList.value.splice(index, 0, image)
}
</script>
<style lang="scss" scoped>
// 整个消息的容器
.message-line {
  display: flex;
  flex-direction: row;
  margin: 24rpx 0;
}

.message-line-self {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: flex-end;
}

.message-line-other {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

// 消息内容的容器
.message-content-container {
  padding: 24rpx;
  border-radius: 16rpx;
  max-width: 75vw;
}

.message-content-container.self {
  background: #007aff;
  padding: 24rpx;
}

.message-content-container.other {
  background: #f3f3f5;
  padding: 24rpx;
}

.message-content-text {
  flex: 1;
  display: flex;
  flex-direction: row;
  font-size: 28rpx;
  flex-wrap: wrap;
}

.message-content-text.self {
  color: #ffffff;
  justify-content: flex-end;
}

.message-content-text.other {
  color: #000000;
  justify-content: flex-start;
}

.message-content-files {
  flex: 1;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
}

.message-content-files.self {
  justify-content: flex-end;
}

.message-content-files.other {
  justify-content: flex-start;
}

.image-container {
  width: 100rpx;
  height: 100rpx;
  margin: 6rpx 6rpx;
  position: relative;
}

.message-avatar {
  image {
    width: 80rpx;
    height: 80rpx;
  }
}

.spin-mask {
  width: 100%;
  height: 100%;
  position: absolute; /* 设置绝对定位 */
  top: 0; /* 设置绝对定位之后还要设置top不然元素还是在图片下面 */
  background: rgba(0, 0, 0, 0.5);
  color: white;
  font-size: 30px;
  /*box-sizing: border-box;*/
  /*padding: 45px 0;*/
  line-height: 120px;
  transition: opacity 0.3s ease;
}

@keyframes spinner {
  0% {
    transform: translate3d(-50%, -50%, 0) rotate(0deg);
  }
  100% {
    transform: translate3d(-50%, -50%, 0) rotate(360deg);
  }
}

.spin::before {
  animation: 1.5s linear infinite spinner;
  animation-play-state: inherit;
  border: solid 15rpx #cfd0d1;
  border-bottom-color: #1c87c9;
  border-radius: 50%;
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate3d(-50%, -50%, 0);
  will-change: transform;
}
</style>
