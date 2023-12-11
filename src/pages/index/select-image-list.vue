<template>
  <view class="selector">
    <view class="add-container" @click="handleSelectImageClick">
      <image src="@/static/plus_black.svg" />
    </view>
    <view v-for="item in state.imageList">
      <view class="image-container">
        <image class="select-image" :src="item.imgPath" mode="aspectFit" @click="handlePreviewImage(item)" />
        <image class="delete-btn" src="@/static/clear.svg" @click="handleDeleteImage(item)" />
      </view>
    </view>
  </view>
</template>
<script setup lang="ts">
import { reactive, defineExpose, defineEmits, watch, ref } from "vue"
import { AttachTypeEnum } from "./config"

const emits = defineEmits(["imageListChange"])
// 图片压缩的质量，大于该值则压缩，否则直接使用，单位为字节，1000byte=1kb，-1则不压缩图片
const maxCompressSize = ref(5000)
// 最多选择图片数量，默认9张
const maxImageListLength = ref(9)
// 非h5平台压缩图片质量
const nh5CompressQuality = ref(70)
// h5压缩图片质量
const h5CompressQuality = ref(0.3)
const userStore = reactive({
  userWsid: ""

})

interface SelectImageListProps {
  maxCompressSize: number
  maxImageListLength: number
  nh5CompressQuality: number
  h5CompressQuality: number
}

withDefaults(defineProps<SelectImageListProps>(), {
  maxCompressSize: 5000,
  maxImageListLength: 9,
  nh5CompressQuality: 70,
  h5CompressQuality: 0.3
})

export interface SelectType {
  type: AttachTypeEnum // 选择文件类型
  imgPath: string
}

interface State {
  imageList: AnyObject[]
}

const state = reactive<State>({
  imageList: []
})

watch(
  state.imageList,
  () => {
    console.log("imagelist change", state.imageList)
    emits("imageListChange", state.imageList)
  },
  {
    immediate: true
  }
)

// 压缩图片，在H5上面使用自定义的压缩图片方法，在其他平台使用uniapp自带的uni.compressImage压缩
const compressImageImpl = (image: AnyObject, callback: Function) => {
  let imagePath = image.path
  // #ifdef H5
  // 压缩质量0.3
  compressImageForH5(imagePath, h5CompressQuality.value, image.type, res => {
    callback(res)
  })
  // #endif
  // #ifndef H5
  uni.compressImage({
    src: imagePath,
    quality: nh5CompressQuality.value,
    success: res => {
      callback(res)
    },
    fail: error => {
      uni.showToast({ title: "压缩图片错误" })
    }
  })
  // #endif
}

// 选择图片
const chooseImageImpl = () => {
  uni.chooseImage({
    count: 1,
    success: res => {
      if (res.errMsg === "chooseImage:ok" && res.tempFilePaths && res.tempFilePaths.length) {
        let image = res.tempFiles[0]
        // 大于5kb则压缩图片
        if (image.size > maxCompressSize.value) {
          // 压缩图片
          compressImageImpl(res.tempFiles[0], compressedImage => {
            state.imageList.push({ type: AttachTypeEnum.IMAGE, imgPath: compressedImage })
          })
        } else {
          state.imageList.push({ type: AttachTypeEnum.IMAGE, imgPath: image.path })
        }
      } else {
        uni.showToast({ title: "选择图片失败" })
      }
    },
    fail: error => {
      uni.showToast({ title: "选择图片失败：" + error })
    }
  })
}

// 大于最大选择数提示不能选择，这里需要加上第一个添加按钮所以加1
const handleSelectImageClick = () => {
  if (state.imageList.length >= maxImageListLength.value + 1) {
    uni.showToast({ title: "最多选择9张图片" })
    return
  } else {
    chooseImageImpl()
  }
}

// 预览图片
const handlePreviewImage = (image: SelectType) => {
  let index = state.imageList.indexOf(image)
  let imgArr = state.imageList.map(img => {
    if (img.type === AttachTypeEnum.IMAGE) {
      return img.imgPath
    } else {
    }
    // 其他类型文件的预览
  })
  uni.previewImage({
    urls: imgArr,
    current: imgArr[index]
  })
}

const handleDeleteImage = (item: SelectType) => {
  state.imageList.splice(state.imageList.indexOf(item), 1)
}

const getImageList = () => {
  return state.imageList.slice(1, state.imageList.length + 1)
}

const resetImageList = () => {
  if (state.imageList && state.imageList.length) {
    state.imageList.splice(0, state.imageList.length)
  }
}


/**
 * h5的图片压缩方法，在H5中使用document，其他端无法获取document对象
 * @param imgSrc 图片的地址
 * @param scale 压缩比例0-1，值越小压缩越大，图片质量越差
 * @param type 压缩的图片类型，这里默认使用image/jpeg，如果为base64则直接传回base64字符串
 * @param callback
 */
const compressImageForH5 = (imgSrc: string, scale: number, type: string, callback: Function) => {
  const img = new Image()
  img.src = imgSrc
  img.onload = function() {
    const h = img.height // 默认按比例压缩
    const w = img.width
    let canvas: HTMLCanvasElement | null = document.createElement("canvas")
    const ctx = canvas.getContext("2d")
    const width = document.createAttribute("width")
    width.nodeValue = w.toString()
    const height = document.createAttribute("height")
    height.nodeValue = h.toString()
    canvas.setAttributeNode(width)
    canvas.setAttributeNode(height)

    if (ctx === null || ctx === undefined) {
      // TODO
    } else {
      ctx.fillStyle = "#fff"
      ctx.fillRect(0, 0, canvas.width, canvas.height)
      ctx.drawImage(img, 0, 0, w, h)
      // png如果使用自带的image/png压缩则没有效果
      // scale为压缩质量，在0-1之间，越大越没效果
      const base64 = canvas.toDataURL("image/jpeg", scale) // 压缩比例
      // 销毁canvas
      canvas = null
      if (type == "base64") {
        callback(base64)
      } else {
        let blob = base64ToBlob(base64)
        let blobUrl = window.URL.createObjectURL(blob) //blob地址
        console.log("compressed imagge", img, base64.length)
        callback(blobUrl)
      }
    }
  }
}

/**
 * base64图片转为blob对象
 * @param base64 图片的base64字符串
 */
const base64ToBlob = base64 => {
  let arr = base64.split(",")
  let mime = arr[0].match(/:(.*?);/)[1]
  let bstr = atob(arr[1])
  let n = bstr.length
  let u8arr = new Uint8Array(n)
  while (n--) {
    u8arr[n] = bstr.charCodeAt(n)
  }
  return new Blob([u8arr], {
    type: mime
  })
}

defineExpose({
  getImageList,
  resetImageList
})
</script>
<style lang="scss" scoped>
page {
  background: #fff;
}

.selector {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  margin: 0 18rpx;
}

.add-container {
  width: calc((100vw - 36rpx) / 5 - 24rpx);
  margin: 12rpx;
  height: 140rpx;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #f6f6f6;
  border-radius: 8rpx;

  image {
    width: 40rpx;
    height: 40rpx;
  }
}

.image-container {
  width: calc((100vw - 36rpx) / 5 - 24rpx);
  margin: 12rpx;
  height: 140rpx;
  display: inline-block;
  border-radius: 8rpx;
  position: relative;
  background-color: rgb(0, 0, 0, 0.05);

  .select-image {
    width: 100%;
    height: 100%;
  }

  .delete-btn {
    width: 40rpx;
    height: 40rpx;
    position: absolute;
    top: 0;
    right: 0;
  }
}
</style>
