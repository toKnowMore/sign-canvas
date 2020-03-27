<template>
  <div class="sign-canvas">
    <van-overlay :show="show" @click=" false">
      <div class="wrapper" @click.stop>
        <div ref="canvasHw" class="overlay-content">
          <div class="overlay-title">请签名</div>
          <canvas
            ref="canvas"
            canvas-id="signCanvas"
            @touchstart="touchStart"
            @touchmove="touchMove"
            @touchend="touchEnd"
            @mousedown="mouseDown"
            @mousemove="mouseMove"
            @mouseup="mouseUp"/>
          <div class="contents-buttons">
            <div class="contents-button" @click="cancleSign">取消</div>
            <div class="contents-button" @click="clearCanvas">清除</div>
            <div class="contents-button" @click="saveCanvas">确定</div>
          </div>
        </div>
      </div>
    </van-overlay>
  </div>
</template>

<script>
var canvas = null
var canvasTxt = null
var points = []
export default {
  name: 'SignCanvas',
  props: {
    show: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      client: {},
      startX: 0,
      startY: 0,
      moveY: 0,
      moveX: 0,
      isDown: false
    }
  },
  mounted() {
    this.initCanvas()
  },
  methods: {
    initCanvas() {
      canvas = this.$refs.canvas// 指定canvas
      canvas.width = window.screen.availWidth - 80
      canvas.height = 300
      canvasTxt = canvas.getContext('2d')// 设置2D渲染区域
      canvasTxt.lineWidth = 3 // 设置线的宽度
      canvasTxt.lineCap = 'round'
      canvasTxt.lineJoin = 'round'
    },
    // mobile
    touchStart(ev) {
      ev = ev || event
      ev.preventDefault()
      if (ev.touches.length === 1) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        this.startX = obj.x
        this.startY = obj.y
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.closePath()
        canvasTxt.stroke()
        points.push(obj)
      }
    },
    touchMove(ev) {
      ev = ev || event
      ev.preventDefault()
      if (ev.touches.length === 1) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.stroke()
        canvasTxt.closePath()
        this.startX = obj.x
        this.startY = obj.y
        points.push(obj)
      }
    },
    touchEnd(ev) {
      ev = ev || event
      ev.preventDefault()
      if (ev.touches.length === 1) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.stroke()
        canvasTxt.closePath()
        points.push(obj)
      }
    },
    // pc
    mouseDown(ev) {
      ev = ev || event
      ev.preventDefault()
      if (ev.touches.length === 1) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        this.startX = obj.x
        this.startY = obj.y
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.stroke()
        canvasTxt.closePath()
        points.push(obj)
        this.isDown = true
      }
    },
    mouseMove(ev) {
      ev = ev || event
      ev.preventDefault()
      if (this.isDown) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.stroke()
        canvasTxt.closePath()
        this.startY = obj.y
        this.startX = obj.x
        points.push(obj)
      }
    },
    mouseUp(ev) {
      ev = ev || event
      ev.preventDefault()
      if (ev.touches.length === 1) {
        const obj = {
          x: ev.touches[0].pageX - canvas.offsetLeft,
          y: ev.touches[0].pageY - canvas.offsetTop
        }
        canvasTxt.beginPath()
        canvasTxt.moveTo(this.startX, this.startY)
        canvasTxt.lineTo(obj.x, obj.y)
        canvasTxt.stroke()
        canvasTxt.closePath()
        points.push(obj)
        points.push({ x: -1, y: -1 })
        this.isDown = false
      }
    },
    cancleSign() {
      this.clearCanvas()
      this.$emit('cancle')
    },
    // 重写
    clearCanvas() {
      canvasTxt.clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)
      points = []
      this.$emit('clear')
    },
    // 确定签名
    async saveCanvas() {
      const file = {}
      file.type = 'image/png'
      file.name = 'sign.png'
      file.content = this.$refs.canvas.toDataURL()
      const imgFile = await this.dataURLToBlob(file)
      this.$emit('save', imgFile)
    },
    dataURLToBlob(file) {
      var arr = file.content.split(',')
      const bstr = atob(arr[1])
      let n = bstr.length
      const u8arr = new Uint8Array(n)
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n)
      }
      return new File([u8arr], file.name, {
        type: file.type
      })
    }
  }
}
</script>

<style scoped lang="scss">
.wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 0 40px;
}

.overlay-content {
  border: 1px solid #ededed;
  background-color: white;
  border-radius: 20px;
  width: 100%;
  .overlay-title {
    text-align: center;
    font-size: 16px;
    padding: 10px 0;
    font-weight: bold;
    border-bottom: 1px solid #ededed;
  }
  .contents-buttons {
    display: flex;
    justify-content: space-between;
    padding: 5px 0;
    border-top: 1px solid #ededed;
    .contents-button {
      padding: 10px 0;
      text-align: center;
      flex: 1;
      font-size: 15px;
      color: rgba(37, 127, 255, 1);
    }
  }
}
</style>
