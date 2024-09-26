<template lang="pug">
.wrapper
    #lottie(:style="wrapperStyle" v-show="selectedFile")
    .handlers 
        el-upload(
            v-show="!selectedFile"
            v-model:file-list="fileList"
            class="upload-demo"
            :multiple="false"
            :show-file-list="false"
            :auto-upload="false"
            :on-change="handlePreview")
            el-button(type="primary" :icon="DocumentAdd") Choose your lottie file
        el-card.card(v-show="selectedFile && startRecordTime == 0")
            el-form(:model="form" label-width="auto")
                el-form-item(label="Single frame size")
                    el-input(v-model="form.width" placeholder="width" style="max-width: 100px")
                    |&nbsp; * &nbsp;
                    el-input(v-model="form.height" placeholder="height" style="max-width: 100px")
                el-form-item(label="Transprent PNG")
                    el-switch(v-model="form.transprent")
                el-form-item(label="Background" v-if="!form.transprent")
                    el-color-picker(v-model="form.bg" show-alpha)
                el-form-item(label="FPS")
                    el-input(v-model="form.fps")
            el-button(type="primary" :icon="Refresh" style="margin: 10px" @click="onSubmit") Start convert
            el-button(type="plain" :icon="ArrowLeft" style="margin: 10px" @click="clear") Choose another
        el-card.card(v-show="startRecordTime != 0")
            | Exporting, please wait....
</template>

<script setup lang="ts">
import { DocumentAdd, Refresh, ArrowLeft } from '@element-plus/icons-vue'
import type { UploadProps, UploadUserFile } from 'element-plus'
import { computed, reactive, ref } from 'vue'
import lottie from 'lottie-web'

const selectedFile = ref(false)
const fileList = ref<UploadUserFile[]>([])
const frames = ref<string[]>([])
const lottieFile = ref({})

let animation: any = null
let lastFrameTime = 0
let frameCount = 0
let startRecordTime = ref(0)

async function handlePreview(file: any) {
    selectedFile.value = true
    lottieFile.value = JSON.parse(await file.raw.text())
    animation = lottie.loadAnimation({
        container: document.getElementById('lottie') as Element, // the dom element that will contain the animation
        renderer: 'canvas',
        loop: true,
        autoplay: true,
        animationData: lottieFile.value
    })
    console.log(animation)
}

function replayAniAndRecord() {
    // 如果有之前的动画，先停止
    if (animation) {
        animation.goToAndPlay(0)
    }
    startRecordTime.value = +new Date()
    // 开始捕获帧
    requestAnimationFrame(captureFrames)
}

function captureFrames(timestamp: number) {
    if (!animation || !animation.isLoaded) return

    if (timestamp - lastFrameTime >= 1000 / form.fps) {
        lastFrameTime = timestamp
        const canvas = document
            .getElementById('lottie')
            ?.getElementsByTagName('canvas')[0] as HTMLCanvasElement
        const dataUrl = canvas.toDataURL('image/png')

        // 将当前帧的图像添加到数组中
        frames.value.push(dataUrl)
        frameCount++
    }

    // 如果动画还在播放，继续捕获
    // console.log(+new Date() - startRecordTime, animation.duration)
    if (
        +new Date() - startRecordTime.value <
        (animation.totalFrames / animation.frameRate) * 1000
    ) {
        requestAnimationFrame(captureFrames)
    } else {
        // 动画结束时处理导出
        exportFrames()
        startRecordTime.value = 0
        // console.log(frames)
    }
}

function exportFrames() {
    const combinedCanvas = document.createElement('canvas')
    const context = combinedCanvas.getContext('2d')

    const totalFrames = frames.value.length
    combinedCanvas.width = +form.width * totalFrames // 设置总宽度
    combinedCanvas.height = +form.height // 高度保持不变

    frames.value.forEach((frameDataUrl, index) => {
        const img = new Image()
        img.src = frameDataUrl
        img.onload = () => {
            // 计算当前帧的绘制位置
            const x = index * +form.width // 每帧的起始 x 坐标
            context?.drawImage(img, x, 0, +form.width, +form.height) // 绘制帧
            // 如果是最后一帧，导出 PNG
            if (index === frames.value.length - 1) {
                const finalDataUrl = combinedCanvas.toDataURL('image/png')
                downloadPNG(finalDataUrl)
            }
        }
    })
}

function downloadPNG(dataUrl: string) {
    const a = document.createElement('a')
    a.href = dataUrl
    a.download = 'lottie-animation.png'
    a.click()
}

const form = reactive({
    width: '300',
    height: '300',
    transprent: true,
    bg: '',
    fps: 30
})

const onSubmit = () => {
    console.log('submit!', form)
    replayAniAndRecord()
}

function clear() {
    selectedFile.value = false
    frames.value = []
    lottie.destroy()
    document.getElementById('lottie').innerHTML = ''
}

const wrapperStyle = computed(() => ({
    width: (form.width || 300) + 'px',
    height: (form.height || 300) + 'px',
    background: form.bg
}))
</script>

<style scoped>
.wrapper {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
}
.card {
    max-width: 600px;
    margin-top: 30px;
}
</style>
