<template>
  <div class="vue-file-viewer" id="vue-file-viewer">
    <div class="banner" v-if="shoHead || !hidden">
      <div class="file-select">
        <button
          type="button"
          style="margin-right: 20px"
          @click.stop="isUrl = !isUrl"
        >
          【点击切换】{{ isUrl ? '上传预览' : '输入网址' }}
        </button>
        <div class="overlay">
          <span v-if="isUrl" style="white-space: pre;">
            <div style="margin-bottom: 10px;">
              <select
                v-model="selectedFileType"
                @change="handleFileTypeChange"
                style="height: 30px; border-radius: 6px; margin-right: 10px; padding: 0 5px;"
              >
                <option value="">请选择文件类型</option>
                <option value="docx">Word文档</option>
                <option value="pptx">PPT文档</option>
                <option value="pdf">PDF文档</option>
                <option value="png">PNG图片</option>
                <option value="mp4">MP4视频</option>
                <option value="mp3">MP3音频</option>
              </select>
            </div>
            <input
              type="text"
              v-model="inputUrl"
              placeholder="请输入浏览文件地址"
            />
            <button type="button" @click.stop="loadFromUrl(inputUrl, true)">
              预览
            </button>
            <button
              type="button"
              @click.stop="clearSelection"
              style="background: #f5f5f5; color: #333; margin-left: 5px;"
            >
              清除
            </button>
          </span>
          <span v-else>
            <input type="file" @change="handleChange" />
          </span>
        </div>
      </div>
    </div>
    <div v-show="!loading && showScale" class="ctrol_btn">
      <span>
        <span class="scale_add" @click="scaleBtn('add')"></span>
        <span class="scale_reduce" @click="scaleBtn('reduce')"></span>
      </span>
      <span style="padding-right:15px;color:rgba(255,255,255,0.3)">|</span>
      <span
        class="download"
        @click="fileDownload(inputUrl || iframeFile, uploadFileName)"
      >
        下载
      </span>
    </div>
    <div style="height: 100%;width: 100%;">
      <div v-show="loading" class="loading">正在加载中，请耐心等待...</div>
      <div
        v-show="!loading"
        ref="output"
        class="output"
        style="height: 100%;width: 100%;overflow: auto;transform-origin: top left;background: rgb(65, 55, 55);"
        :style="{
          transform: `scale(${clientZoom})`,
          height: (1 / clientZoom) * 100 + '%',
          width: (1 / clientZoom) * 100 + '%'
        }"
      ></div>
    </div>
  </div>
</template>

<script>
import { getExtend, readBuffer, render, fileDownload } from './util'
import EventBus from './util/EventBus'
import { typeInfo } from './renders'
import renders from './renders'
import { parse } from 'qs'
import axios from 'axios'

/**
 * 支持嵌入式显示，基于postMessage支持跨域
 * 示例代码：
 *
 */
export default {
  name: 'VueFileViewer',
  props: {
    // 上传的地址
    fileUrl: {
      type: [String, Object],
      default: ''
    },
    // 是否显示头部
    shoHead: {
      type: Boolean,
      default: false
    },
    // 是否开启使用内置的微软文档在线访问接口
    useOfficeMicroOnline: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      // 是否默认为输入网址
      isUrl: true,
      // input输入的url
      inputUrl: '',
      // 上传文件名
      uploadFileName: '',
      // 通过iframe传入的文件
      iframeFile: '',
      // 加载状态跟踪
      loading: false,
      // 是否开启放大缩小按钮
      showScale: false,
      // 隐藏头部，当基于消息机制渲染，将隐藏
      hidden: false,
      // 安全宽度（低于此内容无法展示全）
      safeWith: 1400,
      // 当前文档的缩放比例
      clientZoom: 1,
      // 用于存储选择的文件类型
      selectedFileType: '',
      // 文件类型示例URL配置
      fileTypeExamples: {
        mp3: window.location.origin + '/fileTest/mp3.mp3',
        mp4: window.location.origin + '/fileTest/mp4.mp4',
        pdf: window.location.origin + '/fileTest/pdf.pdf',
        png: window.location.origin + '/fileTest/pic.png',
        pptx: window.location.origin + '/fileTest/ppt.pptx',
        docx: window.location.origin + '/fileTest/word.docx'
      }
    }
  },
  mounted() {
    // 作为iframe使用时，允许使用预留的消息机制发送二进制数据，必须在url后添加?name=xxx.xxx&from=xxx
    const { from, name, fileUrl, shoHead, useOfficeMicroOnline } = parse(
      location.search.substring(1)
    )
    if (from) {
      window.addEventListener('message', (event) => {
        const { origin, data: blob } = event
        if (origin === from && blob instanceof Blob) {
          // 构造响应，自动渲染
          const file = new File([blob], name, {})
          this.iframeFile = file
          this.loadFromBlob(file)
        }
      })
    }
    // 作为iframe使用时，允许通过链接传参获取文件链接数据
    if (fileUrl) {
      this.iframeFile = fileUrl
      this.loadFromUrl(fileUrl, Boolean(shoHead), Boolean(useOfficeMicroOnline))
    }
    // 作为组件使用时，允许接收不同格式的文件数据（链接 or file）
    if (this.fileUrl) {
      typeof this.fileUrl === 'string'
        ? this.loadFromUrl(
            this.fileUrl,
            this.shoHead,
            this.useOfficeMicroOnline
          )
        : this.loadFromBlob(this.fileUrl)
    }
    // 窗体大小改变时自动计算缩放比例
    window.onload = window.onresize = () => {
      this.bodyScale()
    }

    EventBus.$on('fileLoaded', (event) => {
      this.$emit('fileLoaded', event)
      // Using postMessage to notify parent window
      window.parent.postMessage({ type: 'fileLoaded', data: event }, '*')
    })
  },
  methods: {
    fileDownload,
    // 设置缩放比例
    scaleBtn(type) {
      if (this.clientZoom <= 0.1 && type !== 'add') return
      type === 'add' ? (this.clientZoom += 0.1) : (this.clientZoom -= 0.1)
    },
    // 自动缩放比例
    bodyScale() {
      const devicewidth = document.getElementById('vue-file-viewer').clientWidth
      const scale = devicewidth / this.safeWith // 分母——设计稿的尺寸
      this.clientZoom = scale
    },
    // 从url加载
    loadFromUrl(url, shoHead = false, useOfficeMicroOnline = false) {
      // 校验链接是否合法
      if (!url) return

      this.hidden = !shoHead //隐藏头部
      this.loading = true
      this.inputUrl = url
      // 要预览的文件地址
      this.uploadFileName = url.substr(url.lastIndexOf('/') + 1)
      // 取得扩展名并统一转小写兼容大写
      const extend = getExtend(this.uploadFileName).toLowerCase()
      // 判断是否为office文件
      const isOffice = typeInfo.office.find((item) => item.indexOf(extend) > -1)
      // 判断是否需要使用外部微软第三方office在线浏览的方式
      if (useOfficeMicroOnline && isOffice) {
        // 展示微软第三方office在线浏览
        renders['officeOnline'](
          url,
          this.$refs.output,
          'officeOnline',
          this.uploadFileName
        ).finally(() => {
          this.loading = false
        })
        return
      }
      // 内部渲染的方式
      // 拼接iframe请求url
      axios({
        url,
        method: 'get',
        responseType: 'blob'
      })
        .then(({ data }) => {
          const file = new File([data], this.uploadFileName, {})
          this.handleChange({ target: { files: [file] } })
        })
        .finally(() => {
          this.loading = false
        })
        .catch(() => {
          console.error('文件下载失败')
          // 展示文件不存在
          renders['notFind'](
            url,
            this.$refs.output,
            extend,
            this.uploadFileName
          )
        })
    },
    // 从file文件流加载
    loadFromBlob(file) {
      this.hidden = true //隐藏头部
      this.handleChange({ target: { files: [file] } })
    },
    // 监听上传事件获取文件信息，并处理文件
    async handleChange(e) {
      this.loading = true
      try {
        const [file] = e.target.files
        const arrayBuffer = await readBuffer(file)
        this.loading = false
        await this.displayResult(arrayBuffer, file)
      } catch (e) {
        console.error(e)
      } finally {
        this.loading = false
      }
    },
    // 文件信息处理，对应文件渲染方法
    displayResult(buffer, file) {
      // 取得文件名
      const { name } = file
      this.uploadFileName = name
      document.title = name
      // 取得扩展名并统一转小写兼容大写
      const extend = getExtend(name).toLowerCase()
      // 不支持的类型不显示缩放按钮
      if (!renders[extend]) {
        this.showScale = false
      } else {
        this.showScale = true
      }
      // 生成新的dom
      const node = document.createElement('div')
      // 清空容器里的元素
      this.$refs.output.innerHTML = ''
      // 容器追加新的子元素
      const child = this.$refs.output.appendChild(node)
      // 调用对应渲染方法进行渲染
      return new Promise((resolve, reject) =>
        render(buffer, child, extend, name)
          .then(() => {
            // 渲染结束调整缩放比例
            this.$nextTick(() => {
              // 对音频、视频和图片类型特殊处理缩放比例
              const isAudio = typeInfo.audio && typeInfo.audio.includes(extend)
              const isVideo = typeInfo.video && typeInfo.video.includes(extend)
              const isImage = typeInfo.image && typeInfo.image.includes(extend)

              if (isAudio || isVideo) {
                // 对于音频、视频，固定缩放比例为1，不根据屏幕宽度自动缩放
                this.clientZoom = 1
              } else if (isImage) {
                //对于图片类型，固定缩放比例为0.7，不根据屏幕宽度自动缩放
                this.clientZoom = 0.7
              } else {
                // 其他类型使用正常的缩放计算
                this.bodyScale()
              }
            })
            resolve()
          })
          .catch(reject)
      )
    },
    // 处理文件类型选择的逻辑
    handleFileTypeChange() {
      if (
        this.selectedFileType &&
        this.fileTypeExamples[this.selectedFileType]
      ) {
        // 根据选择的文件类型填充对应的示例URL
        this.inputUrl = this.fileTypeExamples[this.selectedFileType]

        // 直接自动触发预览
        this.loadFromUrl(this.inputUrl, true)
      } else {
        // 如果没有选择文件类型或没有对应的示例URL，清空输入框
        this.inputUrl = ''
      }
    },

    // 清除选择和输入框
    clearSelection() {
      this.selectedFileType = ''
      this.inputUrl = ''
      // 清空输出区域
      if (this.$refs.output) {
        this.$refs.output.innerHTML = ''
      }
    }
  }
}
</script>

<style lang="scss" src="./index.scss" scoped></style>
