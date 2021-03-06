<template lang="pug">
  div(:class="$style.inner")
    RealVolume(:volume="volume")
    div {{recordingTime}}s
    v-btn(@click="leftAudioData.length===0?record():stopRecord()")
      | {{leftAudioData.length===0?'录制':'停止录制'}}
    v-btn(@click="playAudio") 播放录音
    v-btn(@click="downloadToLocal") 保存为本地音乐文件
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import RealVolume from '@/components/real-volume.vue'
import { downloadToLocal, playAudio } from '@/utils/record'

let timer = 0

@Component({
  components: { RealVolume }
})
export default class SoundRecord2 extends Vue {
  leftAudioData: Array<Float32Array> = []// 左声道音频数据
  rightAudioData: Array<Float32Array> = []// 右声道音频数据
  mediaStream!: MediaStream// 媒体流
  currentFile: File | null = null// 当前录制音频文件流
  jsNode!: ScriptProcessorNode
  mediaNode!: MediaStreamAudioSourceNode
  recordingTime = 0
  // 实时音量
  volume = 0

  /**
   * 录制
   * @returns {Promise<void>}
   */
  async record (): Promise<void> {
    try {
      // 获取麦克风媒体流
      const stream = this.mediaStream = await navigator.mediaDevices.getUserMedia({
        audio: {
          sampleRate: 48000,
          channelCount: 2
        },
        video: false
      })

      // 通过WebAudio保存录音
      const ac = new AudioContext()
      // 通过媒体流创建一个audioNode
      const mediaNode = this.mediaNode = ac.createMediaStreamSource(stream)

      // 绑定createScriptProcessor的this为AudioContext，然后创建一个处理了音频的节点
      const creator = ac.createScriptProcessor.bind(ac)
      const jsNode = this.jsNode = creator(16384, 2, 2)// 设置的更小的话会造成有杂音

      // 连接到AudioContext
      jsNode.connect(ac.destination)
      // audioNode连接到jsNode
      mediaNode.connect(jsNode)
      // 添加音频流入事件
      jsNode.onaudioprocess = (e: AudioProcessingEvent) => {
        const audioBuffer = e.inputBuffer
        const leftData = audioBuffer.getChannelData(0)
        const rightData = audioBuffer.getChannelData(1)
        this.volume = rightData[rightData.length - 1] + 1
        // 这里有个坑，如果不进行深拷贝的话，录制出来的音频会有问题
        this.leftAudioData.push(leftData.slice(0))
        this.rightAudioData.push(rightData.slice(0))
      }
      timer = setInterval(() => {
        this.recordingTime++
      }, 1000)
    } catch (e) {
      console.log(e)
      if (e.name === 'TypeError') {
        alert('当前环境不支持视频通话')
      } else if (e.name === 'NotAllowedError') {
        alert('请允许使用麦克风')
      } else if (e.name === 'AbortError') {
        console.log('中止')
      } else if (e.name === 'NotFoundError') {
        console.log('找不到')
      } else if (e.name === 'OverConstrainedError') {
        console.log('设备无法满足要求')
      } else if (e.name === 'SecurityError') {
        console.log('由于安全原因，被禁止')
      }
    }
  }

  stopRecord (): void {
    const { mediaStream } = this
    // 获取所有的媒体通道并停止他们
    const tracks = mediaStream.getTracks()

    clearInterval(timer)

    tracks.forEach(track => {
      track.stop()
    })
    // 停止录音
    this.jsNode.disconnect()
    this.mediaNode.disconnect()

    // 合并数据
    const left = mergeArray(this.leftAudioData)
    const right = mergeArray(this.rightAudioData)
    const audioData = mergedProvincialHighway(left, right)
    this.currentFile = createAudioFile(audioData)
    this.leftAudioData = []
    this.rightAudioData = []
  }

  /**
   * 下载至本地
   */
  async downloadToLocal (): Promise<void> {
    try {
      const { currentFile } = this
      await downloadToLocal(currentFile as File, '请先录制完成，再下载')
    } catch (e) {
      alert(e)
    }
  }

  /**
   * 播放当前录制的音频
   */
  async playAudio (): Promise<void> {
    try {
      const { currentFile } = this
      await playAudio(currentFile as File, '请先录制完成，再播放')
    } catch (e) {
      alert(e)
    }
  }
}

/**
 *交叉合并两个声道
 * @param {Float32Array} left 左声道
 * @param {Float32Array} right 右声道
 */
function mergedProvincialHighway (left: Float32Array, right: Float32Array): Float32Array {
  const length = left.length + right.length
  const data = new Float32Array(length)
  for (let i = 0; i < left.length; i++) {
    const j = i * 2
    data[j] = left[i]
    data[j + 1] = right[i]
  }
  return data
}

/**
 * 根据合成的录音数据创建文件
 * @param {Float32Array} audioData 合成的数据
 * @returns {File} 创建的文件
 */
function createAudioFile (audioData: Float32Array) {
  const bufferLength = audioData.length * 2 + 44
  const buffer = new ArrayBuffer(bufferLength)
  const view = new DataView(buffer)
  let index = 44
  // 下面的操作是创建文件头
  writeUTFBytes(view, 0, 'RIFF')
  // RIFF块长度
  view.setUint32(4, 44 + audioData.length * 2, true)
  // RIFF类型
  writeUTFBytes(view, 8, 'WAVE')
  // 格式块标识符
  // FMT子块
  writeUTFBytes(view, 12, 'fmt ')
  // 格式块长度
  view.setUint32(16, 16, true)
  // 样本格式（原始）
  view.setUint16(20, 1, true)
  // 立体声（2声道）
  view.setUint16(22, 2, true)
  // 采样率
  view.setUint32(24, 44100, true)
  // 字节率（采样率*块对齐）
  view.setUint32(28, 44100 * 2, true)
  // 块对齐（通道数*每个样本字节）
  view.setUint16(32, 2 * 2, true)
  // 每个样本位数
  view.setUint16(34, 16, true)
  // 数据子块
  // 数据块标识符
  writeUTFBytes(view, 36, 'data')
  // 数据块长度
  view.setUint32(40, audioData.length * 2, true)

  // 下面的操作是填入数据
  for (const data of audioData) {
    view.setInt16(index, data * (0x7FFF * 1), true)
    index += 2
  }
  return new File(
    [buffer],
    (new Date()).toISOString().replace('T', ' '),
    {
      type: 'audio/wave'
    }
  )
}

/**
 * 写入字节
 * @param {DataView} view
 * @param {number} offset
 * @param {string} string
 */
function writeUTFBytes (view: DataView, offset: number, string: string) {
  for (let i = 0; i < string.length; i++) {
    view.setUint8(offset + i, string.charCodeAt(i))
  }
}

/**
 * 合并单声道数据片段。因为Float32Array长度不能够动态调节，所以需要根据最终数据合成一个整体的Float32Array
 * @param {Array<Float32Array>} arr 数据片段数组
 * @returns {Float32Array} 返回一个包含所有片段的数组
 */
function mergeArray (arr: Array<Float32Array>): Float32Array {
  const length = arr.length * arr[0].length
  const floatArray = new Float32Array(length)
  let offset = 0
  for (const item of arr) {
    floatArray.set(item, offset)
    offset += item.length
  }
  return floatArray
}
</script>

<style lang="stylus" rel="stylesheet/stylus" module>
.inner
  display grid
  place-content center
  width 100%
  height 100%
  grid-gap 20px
</style>
