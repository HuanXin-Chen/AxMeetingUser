<template>
  <div
    :class="{
      'vue-live2d': true,
      'vue-live2d-on-left': direction === 'left',
      'vue-live2d-on-right': direction === 'right',
    }"
    :style="{
      width: `${live2dWidth}px`,
      height: `${live2dHeight}px`,
    }"
    @mouseover="openLive2dTool"
    @mouseout="closeLive2dTool">
    <div
      v-show="tipShow"
      v-html="tipText"
      :class="{
        'vue-live2d-tip': true,
        'vue-live2d-tip-on-top': tipPosition === 'top',
        'vue-live2d-tip-on-bottom': tipPosition === 'bottom',
      }">
    </div>
    <canvas
      :id="customId"
      v-show="mainShow"
      :class="{
        'vue-live2d-main': true,
        'vue-live2d-main-on-left': direction === 'left',
        'vue-live2d-main-on-right': direction === 'right',
      }"
      :width="live2dWidth"
      :height="live2dHeight">
    </canvas>
    <div v-show="toolShow" class="vue-live2d-tool">
      <span
        v-for="(tool, index) in tools"
        :key="index"
        :class="tool.className"
        v-html="tool.svg"
        @click="tool.click"/>
    </div>
    <div
      v-show="toggleShow"
      @click="openLive2dMain"
      :class="{
        'vue-live2d-toggle': true,
        'vue-live2d-toggle-on-left': direction === 'left',
        'vue-live2d-toggle-on-right': direction === 'right',
      }">
      <span>看板娘</span>
    </div>
  </div>
</template>

<script>
import './lib/live2d.min.js'

import tips from './options/tips'

export default {
  name: 'vue-live2d',
  props: {
    direction: {
      default: 'right',
      validator: function (value) {
        return new Set(['left', 'right']).has(value)
      },
      type: String
    },
    tipPosition: {
      default: 'top',
      validator: function (value) {
        return new Set(['top', 'top']).has(value)
      },
      type: String
    },
    customId: {
      default: 'vue-live2d-main',
      type: String
    },
    apiPath: {
      // 注意：这是我服务器目前部署的 api 服务，若更新服务地址会在 README.md 说明
      default: 'https://evgo2017.com/api/live2d-static-api/indexes',
      type: String
    },
    model: {
      default: () => ['Potion-Maker/Pio', 'school-2017-costume-yellow'],
      type: Array
    },
    homePage: {
      default: 'https://github.com/evgo2017/vue-live2d',
      type: String
    },
    tips: {
      default: () => tips,
      type: Object
    },
    width: {
      default: 0,
      type: Number
    },
    height: {
      default: 0,
      type: Number
    },
    size: {
      default: 255,
      type: Number
    }
  },
  data () {
    return {
      messageTimer: null,
      containerDisplay: {
        tip: false,
        main: true,
        tool: false,
        toggle: false
      },
      tipText: 'vue-live2d 看板娘',
      modelPath: '',
      modelTexturesId: '',
      tools: [{
        className: 'custom-fa-comment',
        svg: '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="currentColor" height="20px" width="20px"><!-- Font Awesome Free 5.15.4 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license/free (Icons: CC BY 4.0, Fonts: SIL OFL 1.1, Code: MIT License) --><path d="M256 32C114.6 32 0 125.1 0 240c0 49.6 21.4 95 57 130.7C44.5 421.1 2.7 466 2.2 466.5c-2.2 2.3-2.8 5.7-1.5 8.7S4.8 480 8 480c66.3 0 116-31.8 140.6-51.4 32.7 12.3 69 19.4 107.4 19.4 141.4 0 256-93.1 256-208S397.4 32 256 32z"/></svg>',
        click: this.showHitokoto
      }]
    }
  },
  mounted () {
    [this.modelPath, this.modelTexturesId] = this.model
    this.loadModel()
    this.$nextTick(() => {
      this.loadEvent()
    })
  },
  computed: {
    live2dWidth () {
      return this.width ? this.width : this.size
    },
    live2dHeight () {
      return this.height ? this.height : this.size
    },
    tipShow () {
      return this.mainShow && this.containerDisplay.tip
    },
    mainShow () {
      return this.containerDisplay.main
    },
    toolShow () {
      return this.mainShow && this.containerDisplay.tool
    },
    toggleShow () {
      return !this.mainShow
    }
  },
  watch: {
    width () {
      this.changeLive2dSize()
    },
    height () {
      this.changeLive2dSize()
    },
    size () {
      this.changeLive2dSize()
    }
  },
  methods: {
    changeLive2dSize () {
      // 针对当前这份 live2d.min.js 来说，更改宽高就是这样。更好的方案是调用重绘方法，但是需要改 lib 源码。
      document.querySelector(`#${this.customId}`).outerHTML = `<canvas id=${this.customId} width="${this.live2dWidth}" height="${this.live2dHeight}" class="vue-live2d-main"></canvas>`
      this.loadModel()
    },
    loadModel () {
      window.loadlive2d(this.customId, `${this.apiPath}/${this.modelPath}/${this.modelTexturesId}.json`)
      console.log(`Live2D 模型 ${this.modelPath}，服装 ${this.modelTexturesId} 加载完成`)
    },
    loadRandModel () {
      this.http({
        url: `${this.apiPath}/models.json`,
        success: (data) => {
          const models = data.filter(({ modelPath }) => modelPath !== this.modelPath)
          const { modelPath, modelIntroduce } = models[Math.floor(Math.random() * models.length)]
          this.modelPath = modelPath
          this.showMessage(`${modelIntroduce}`, 4000)
          this.loadRandTextures(true)
        }
      })
    },
    loadRandTextures (isAfterRandModel = false) {
      this.http({
        url: `${this.apiPath}/${this.modelPath}/textures.json`,
        success: (data) => {
          const modelTexturesIds = data.filter(modelTexturesId => modelTexturesId !== this.modelTexturesId)
          this.modelTexturesId = modelTexturesIds[Math.floor(Math.random() * modelTexturesIds.length)]
          this.loadModel()
          if (!isAfterRandModel) {
            this.showMessage('我的新衣服好看嘛？', 4000)
          }
        }
      })
    },
    showMessage (msg = '', timeout = 6000) {
      if (this.messageTimer) {
        clearTimeout(this.messageTimer)
        this.messageTimer = null
      } else {
        this.containerDisplay.tip = true
      }
      this.tipText = msg
      this.messageTimer = setTimeout(() => {
        this.containerDisplay.tip = false
        this.messageTimer = null
      }, timeout)
    },
    takePhoto () {
      this.showMessage('照好了嘛，留个纪念吖~')
      window.Live2D.captureName = 'photo.png'
      window.Live2D.captureFrame = true
    },
    showHitokoto () {
      this.showMessage('先别急哈，让我想一下呀！')
      this.http({
        url: 'http://8.138.84.212:8080/ai/help',
        success: ({ message }) => {
          this.showMessage(`${message}`)
        }
      })
    },
    openHomePage () {
      open(this.homePage)
    },
    closeLive2dMain () {
      this.containerDisplay.main = false
    },
    openLive2dMain () {
      this.containerDisplay.main = true
    },
    closeLive2dTool () {
      this.containerDisplay.tool = false
    },
    openLive2dTool () {
      this.containerDisplay.tool = true
    },
    loadEvent () {
      for (const event in this.tips) {
        for (const { selector, texts } of this.tips[event]) {
          const dom = selector === 'document' ? document : document.querySelector(selector)
          if (dom == null) {
            continue
          }

          dom.addEventListener(event, () => {
            const msg = texts[Math.floor(Math.random() * texts.length)]
            this.showMessage(msg, 2000)
          })
        }
      }
    },
    http ({ url, success }) {
      const xhr = new XMLHttpRequest()
      xhr.onreadystatechange = function () {
        if (xhr.readyState === 4) {
          if ((xhr.status >= 200 || xhr.status < 300) || xhr.status === 304) {
            success && success(JSON.parse(xhr.response))
          } else {
            console.error(xhr)
          }
        }
      }
      xhr.open('GET', url)
      xhr.send(null)
    }
  }
}
</script>

<style scoped>
/* live2d */
.vue-live2d {
  display: flex;
  position: relative;
  align-items: flex-end;
}
.vue-live2d-on-left {
  flex-direction: row;
}
.vue-live2d-on-right {
  flex-direction: row-reverse;
}
/* live2d-tip */
.vue-live2d-tip {
  box-sizing: border-box;
  position: absolute;
  width: 100%;
  line-height: 1.5rem;
  padding: 15px 20px;
  font-size: .9rem;

  word-break: break-all;
  text-overflow: ellipsis;
  border: 1px solid rgba(224, 186, 140, 0.62);
  border-radius: 12px;
  background-color: rgba(236, 217, 188, 0.5);
  box-shadow: 0 3px 15px 2px rgba(191, 158, 118, 0.2);
  animation: shake 50s ease-in-out 5s infinite;
}
.vue-live2d-tip-on-top {
  top: 0;
}
.vue-live2d-tip-on-bottom {
  bottom: 0;
}
/* live2d-main */
.vue-live2d-main {
  transition: padding .3s ease-in-out;
  cursor: grab;
}
.vue-live2d-main-on-left:hover {
  padding-left: 21px;
}
.vue-live2d-main-on-right:hover {
  padding-right: 21px;
}
/* live2d-tool */
.vue-live2d-tool {
  position: absolute;
  width: 20px;
  bottom: 10px;
  color: #5b6c7d;
  text-align: center;
  cursor: pointer;
  padding: 0 10px;
}
.vue-live2d-tool span {
  line-height: 30px;
}
.vue-live2d-tool span:hover {
  color: #0684bd;
}
/* live2d-toggle */
.vue-live2d-toggle {
  width: 1.5rem;
  bottom: 1rem;
  padding: .3rem 0;
  writing-mode: vertical-lr;
  color: #fff;
  background-color: #fa0;
  font-size: 1rem;
  cursor: pointer;
  right: 0;
}
.vue-live2d-toggle:hover {
  width: 1.7rem;
}
.vue-live2d-toggle-on-left {
  border-radius:  0 .5rem .5rem 0;
}
.vue-live2d-toggle-on-right {
  border-radius:  .5rem 0 0 .5rem;
}
@keyframes shake {
  2% {
    transform: translate(0.5px, -1.5px) rotate(-0.5deg);
  }
  4% {
    transform: translate(0.5px, 1.5px) rotate(1.5deg);
  }
  6% {
    transform: translate(1.5px, 1.5px) rotate(1.5deg);
  }
  8% {
    transform: translate(2.5px, 1.5px) rotate(0.5deg);
  }
  10% {
    transform: translate(0.5px, 2.5px) rotate(0.5deg);
  }
  12% {
    transform: translate(1.5px, 1.5px) rotate(0.5deg);
  }
  14% {
    transform: translate(0.5px, 0.5px) rotate(0.5deg);
  }
  16% {
    transform: translate(-1.5px, -0.5px) rotate(1.5deg);
  }
  18% {
    transform: translate(0.5px, 0.5px) rotate(1.5deg);
  }
  20% {
    transform: translate(2.5px, 2.5px) rotate(1.5deg);
  }
  22% {
    transform: translate(0.5px, -1.5px) rotate(1.5deg);
  }
  24% {
    transform: translate(-1.5px, 1.5px) rotate(-0.5deg);
  }
  26% {
    transform: translate(1.5px, 0.5px) rotate(1.5deg);
  }
  28% {
    transform: translate(-0.5px, -0.5px) rotate(-0.5deg);
  }
  30% {
    transform: translate(1.5px, -0.5px) rotate(-0.5deg);
  }
  32% {
    transform: translate(2.5px, -1.5px) rotate(1.5deg);
  }
  34% {
    transform: translate(2.5px, 2.5px) rotate(-0.5deg);
  }
  36% {
    transform: translate(0.5px, -1.5px) rotate(0.5deg);
  }
  38% {
    transform: translate(2.5px, -0.5px) rotate(-0.5deg);
  }
  40% {
    transform: translate(-0.5px, 2.5px) rotate(0.5deg);
  }
  42% {
    transform: translate(-1.5px, 2.5px) rotate(0.5deg);
  }
  44% {
    transform: translate(-1.5px, 1.5px) rotate(0.5deg);
  }
  46% {
    transform: translate(1.5px, -0.5px) rotate(-0.5deg);
  }
  48% {
    transform: translate(2.5px, -0.5px) rotate(0.5deg);
  }
  50% {
    transform: translate(-1.5px, 1.5px) rotate(0.5deg);
  }
  52% {
    transform: translate(-0.5px, 1.5px) rotate(0.5deg);
  }
  54% {
    transform: translate(-1.5px, 1.5px) rotate(0.5deg);
  }
  56% {
    transform: translate(0.5px, 2.5px) rotate(1.5deg);
  }
  58% {
    transform: translate(2.5px, 2.5px) rotate(0.5deg);
  }
  60% {
    transform: translate(2.5px, -1.5px) rotate(1.5deg);
  }
  62% {
    transform: translate(-1.5px, 0.5px) rotate(1.5deg);
  }
  64% {
    transform: translate(-1.5px, 1.5px) rotate(1.5deg);
  }
  66% {
    transform: translate(0.5px, 2.5px) rotate(1.5deg);
  }
  68% {
    transform: translate(2.5px, -1.5px) rotate(1.5deg);
  }
  70% {
    transform: translate(2.5px, 2.5px) rotate(0.5deg);
  }
  72% {
    transform: translate(-0.5px, -1.5px) rotate(1.5deg);
  }
  74% {
    transform: translate(-1.5px, 2.5px) rotate(1.5deg);
  }
  76% {
    transform: translate(-1.5px, 2.5px) rotate(1.5deg);
  }
  78% {
    transform: translate(-1.5px, 2.5px) rotate(0.5deg);
  }
  80% {
    transform: translate(-1.5px, 0.5px) rotate(-0.5deg);
  }
  82% {
    transform: translate(-1.5px, 0.5px) rotate(-0.5deg);
  }
  84% {
    transform: translate(-0.5px, 0.5px) rotate(1.5deg);
  }
  86% {
    transform: translate(2.5px, 1.5px) rotate(0.5deg);
  }
  88% {
    transform: translate(-1.5px, 0.5px) rotate(1.5deg);
  }
  90% {
    transform: translate(-1.5px, -0.5px) rotate(-0.5deg);
  }
  92% {
    transform: translate(-1.5px, -1.5px) rotate(1.5deg);
  }
  94% {
    transform: translate(0.5px, 0.5px) rotate(-0.5deg);
  }
  96% {
    transform: translate(2.5px, -0.5px) rotate(-0.5deg);
  }
  98% {
    transform: translate(-1.5px, -1.5px) rotate(-0.5deg);
  }
  0%, 100% {
    transform: translate(0, 0) rotate(0deg);
  }
}
</style>
