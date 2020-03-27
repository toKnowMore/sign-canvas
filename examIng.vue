<!-- 模拟考试 -->
<template>
  <div class="examIngPage">
    <div class="app-container">
      <header>
        <div class="header-left">
          <div class="kuai"/>
          <div class="examType">
            {{ currentExam.type | questionType }}
          </div>
        </div>
        <div class="header-center">
          <div class="timeDesc">剩余时间</div>
          <van-count-down
            :time="examTime"
            format="HH:mm:ss"
            @finish="timeFinish">
            <template v-slot="timeData">
              <span class="van-count-down-item">{{ timeData.hours }}</span>
              <span class="van-count-down-item">{{ timeData.minutes }}</span>
              <span class="van-count-down-item">{{ timeData.seconds }}</span>
            </template>
          </van-count-down>
        </div>
        <div class="header-right" @click="toShowModal">
          <van-icon name="apps-o"/>
          <div class="qProgress">
            <span class="currentQs">{{ currentIndex + 1 }}</span>
            <span>/</span>
            <span class="totalQs">{{ listData.length }}</span>
          </div>
        </div>
      </header>
      <section class="container">
        <div class="qTitle">
          <span>{{ currentExam.title }}</span>
          <video v-show="startVideo" ref="myVideo" controls/>
          <canvas v-show="startVideo" ref="myCanvas" width="150" height="230"/>
        </div>
        <div class="selectArea">
          <van-checkbox-group
            v-show="currentExam.type == 1"
            v-model="currentExam.checkList"
            icon-size="20px"
            @change="optionChange">
            <van-checkbox
              v-for="(item, index) in currentExam.options"
              :key="index"
              :name="item.key">{{ item.key }}、{{ item.value }}</van-checkbox>
          </van-checkbox-group>
          <van-radio-group
            v-show="currentExam.type == 2 || currentExam.type == 0"
            v-model="currentExam.select"
            icon-size="20px"
            @change="optionChange">
            <van-radio
              v-for="(item, index) in currentExam.options"
              :key="index"
              :name="item.key">{{ item.key }}、{{ item.value }}</van-radio>
          </van-radio-group>
        </div>
        <div v-if="currentExam.attachType > 0" class="attach">
          <video
            v-if="currentExam.attachType == 2"
            controls="controls"
            preload="auto"
            poster
            muted>
            <source :src="currentExam.attach">
            您的浏览器不支持 video
          </video>
          <img v-if="currentExam.attachType == 1" :src="currentExam.attach">
        </div>
      </section>
      <footer>
        <div class="footerLeft">
          <van-button class="qButton" @click.stop="toLast">上一题</van-button>
          <van-button class="qButton" @click.stop="toNext">下一题</van-button>
        </div>
        <van-button
          :loading="btnLoading"
          type="info"
          class="submit"
          @click.stop="submitExam">
          交卷
        </van-button>
      </footer>
    </div>

    <sign-canvas
      :is-cheat="submitExamEdit.isCheat"
      :show="showCanvas"
      @cancle="cancleCanvas"
      @save="saveCanvas"
      @clear="clearCanvas"/>

    <!-- 题目序列板 -->
    <modal-numbers
      :show="showModal"
      :list="listData"
      :styles="{ top: '50px' }"
      :current-index="currentIndex"
      position="top"
      @click="itemClick"
      @close="toCloseNumbers"/>

    <van-overlay :show="showUploader" @click="showUploader = false">
      <div class="wrapper" @click.stop>
        <van-uploader :after-read="afterReadImg">
          <img v-show="faceImg" :src="faceImg">
          <van-button
            v-show="!faceImg"
            icon="add"
            deletable>请自拍一张照片</van-button>
        </van-uploader>
      </div>
    </van-overlay>
  </div>
</template>

<script>
import modalNumbers from '@/components/modalNumbers'
import signCanvas from '@/components/signCanvas'
import { compressImage } from '@/utils/compressImage'
import {
  examQuestions,
  submitExamQ,
  checkFaceImg
} from '@/api/exam'
import {
  isiOS,
  isAndroid,
  getQueryObject
} from '@/utils'
import {
  setToken, getToken
} from '@/utils/auth'
import { uploadFile } from '@/api/public'
export default {
  components: {
    modalNumbers,
    signCanvas
  },
  filters: {
    isPass: function(val) {
      switch (val) {
        case 0:
          return '通过'
        case 1:
          return '未通过'
        default:
          break
      }
    },
    questionType(val) {
      switch (val) {
        case 0:
          return '单选题'
        case 1:
          return '多选题'
        case 2:
          return '是非题'
        default:
          break
      }
    }
  },
  data() {
    return {
      currentExam: {},
      currentIndex: 0,
      listData: [],
      submitExamEdit: {},
      answers: [],
      examTime: 5400000,
      submit: false,
      btnLoading: false,
      showModal: false,
      examParams: {},
      faceImg: '',
      isLeave: true,

      video: null,
      context: null,
      canvas: null,
      isMonitor: 0,
      timer: null,
      startVideo: false,
      showCanvas: false,
      showUploader: false,
      isSubmit: false
    }
  },
  created(e) {
    // 第一步获取参数，把参数存一下
    // 判断机型，获取题目
    // 安卓就video,ios就弹出上传图片
    // 获取的题目保存一下，当前题目保存一下，题目顺序保存一下
    // const obj = getQueryObject('https://app.xuexiqiangan.com/examVideo?token=074d6423-4e5f-4907-8e04-90e8376f7eff&userId=85&jobId=13&examTime=66')
    const obj = getQueryObject()
    this.submitExamEdit.userId = obj.userId
    this.submitExamEdit.jobId = obj.jobId
    this.examTime = obj.examTime * 60 * 1000
    localStorage.setItem('examTime', this.examTime)
    setToken(obj.token)
  },
  mounted() {
    this.init()
  },
  methods: {
    init() {
      if (getToken()) {
        if (localStorage.getItem('currentIndex')) {
          this.getLocalInfo()
        } else {
          this.getQuestions()
        }
      } else {
        console.log('没有token')
      }
    },
    itemClick(item, index) {
      this.currentIndex = index
      this.showModal = false
      this.buildData()
    },
    toCloseNumbers() {
      this.showModal = false
    },
    parseDate(date) {
      const t = Date.parse(date)
      if (isNaN(t)) {
        return new Date(Date.parse(date.replace(/-/g, '/'))).getTime()
      } else {
        return t
      }
    },
    /**
     * 获取正式考题列表
     */
    getQuestions() {
      examQuestions({ userId: this.submitExamEdit.userId }).then(response => {
        if (response.data) {
          const data = response.data

          this.listData = data.examQuestions

          this.listData.forEach((item, index) => {
            item.checkList = []
            item.select = ''
            item.options = JSON.parse(item.options)
          })

          if (data.isSnap > 0) {
            this.startCamera()
          }

          localStorage.setItem('isSnap', data.isSnap)

          this.submitExamEdit.examResultId = data.examResultId

          this.submitExamEdit.questions = JSON.stringify(data.examQuestions)

          localStorage.setItem('submitExamEdit', JSON.stringify(this.submitExamEdit))

          const nowTime = (new Date()).getTime()

          localStorage.setItem('nowTime', nowTime)

          this.buildData()
        }
      })
    },

    getLocalInfo() {
      const isSnap = localStorage.getItem('isSnap')
      if (isSnap > 0) {
        this.startCamera()
      }

      this.listData = JSON.parse(localStorage.getItem('examQuestions'))
      this.currentIndex = localStorage.getItem('currentIndex')
      this.submitExamEdit = JSON.parse(localStorage.getItem('submitExamEdit'))
      // 继续考试，时间为时间间隔

      const lastTime = localStorage.getItem('nowTime')
      const examLength = localStorage.getItem('examTime')

      const value = localStorage.getItem('currentIndex')
      if (value) {
        this.currentIndex = value - 0
      }
      this.buildData()

      const nowTime = new Date().getTime()

      if (nowTime - lastTime < examLength) {
        this.examTime = nowTime - lastTime
      } else {
        this.timeFinish()
      }
    },

    /**
     * 构建数据
     */
    buildData() {
      this.currentExam = this.listData[this.currentIndex]
      localStorage.setItem('currentIndex', this.currentIndex)
      localStorage.setItem('examQuestions', JSON.stringify(this.listData))
      this.currentExam = JSON.parse(JSON.stringify(this.currentExam))
    },

    toLast() {
      if (this.currentIndex > 0) {
        this.currentIndex--
        this.buildData()
      } else {
        this.$toast('当前是第一题')
      }
    },
    toNext() {
      if (this.currentIndex < this.listData.length - 1) {
        this.currentIndex++
        this.buildData()
      } else {
        this.$toast('当前是最后一题')
      }
    },

    optionChange() {
      this.listData[this.currentIndex] = this.currentExam
    },

    toShowModal() {
      this.showModal = !this.showModal
    },

    submitExam() {
      this.$dialog.confirm({
        title: '提示',
        message: '是否交卷？'
      }).then(() => {
        const isSnap = localStorage.getItem('isSnap')
        if (isSnap > 0 && this.submitExamEdit.isCheat == null) {
          this.isSubmit = true
          if (isAndroid()) {
            this.capture()
          } else if (isiOS()) {
            this.showUploader = true
          }
        } else {
          this.submitExamEdit.isCheat = 0
          this.showCanvas = true
        }
      })
    },

    /**
     * 时间结束
     */
    timeFinish() {
      this.$toast('答题时间到')
      const isSnap = localStorage.getItem('isSnap')
      if (isSnap > 0 && this.submitExamEdit.isCheat == null) {
        this.isSubmit = true
        if (isAndroid()) {
          this.capture()
        } else if (isiOS()) {
          this.showUploader = true
        }
      } else {
        this.submitExamEdit.isCheat = 0
        this.showCanvas = true
      }
    },

    sendSubmitData() {
      this.answers = []
      this.listData.forEach((item, index) => {
        if (item.select) {
          this.answers.push(item.select)
        } else if (item.checkList.length > 0) {
          item.check = item.checkList.join(',')
          this.answers.push(item.check)
        } else {
          this.answers.push('')
        }
      })
      this.submitExamEdit.answers = JSON.stringify(this.answers)
      submitExamQ(this.submitExamEdit).then(response => {
        if (response.data) {
          this.isLeave = false
          localStorage.removeItem('currentIndex')
          localStorage.removeItem('examQuestions')
          localStorage.removeItem('submitExamEdit')
          localStorage.removeItem('nowTime')
          localStorage.removeItem('examTime')
          window.uni.redirectTo({
            url: '/pages/exam/exam-result?isPass=' + response.data.isPass + '&score=' + response.data.score
          })
        }
      })
    },

    startCamera() {
      // 首先清除计时器
      if (this.timer) {
        clearInterval(this.timer)
      }
      if (isAndroid()) {
        this.startVideo = true
        this.initVideo()
      } else if (isiOS()) {
        this.isIosTimer()
      }
    },

    initVideo() {
      this.video = this.$refs.myVideo
      this.canvas = this.$refs.myCanvas
      this.context = this.canvas.getContext('2d')
      if (navigator.mediaDevices.getUserMedia || navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia) {
      // 调用用户媒体设备, 访问摄像头
        this.getUserMedia({ video: { width: 150, height: 200, 'facingMode': 'user' }}, this.success, this.error)
      } else {
        this.$toast('不支持访问用户媒体')
      }
    },

    getUserMedia(constraints, success, error) {
      if (navigator.mediaDevices.getUserMedia) {
        // 最新的标准API
        navigator.mediaDevices.getUserMedia(constraints).then(success).catch(error)
      } else if (navigator.webkitGetUserMedia) {
        // webkit核心浏览器
        navigator.webkitGetUserMedia(constraints, success, error)
      } else if (navigator.mozGetUserMedia) {
        // firfox浏览器
        navigator.mozGetUserMedia(constraints, success, error)
      } else if (navigator.getUserMedia) {
        // 旧版API
        navigator.getUserMedia(constraints, success, error)
      }
    },

    success(stream) {
      // 兼容webkit核心浏览器
      try {
        this.video.srcObject = stream
        this.video.play()
      } catch (err) {
        const CompatibleURL = window.URL || window.webkitURL
        this.video.src = CompatibleURL.createObjectURL(stream)
        this.video.play()
      }
      this.isAndroidTimer()
    },

    error(error) {
      console.log(error)
    },

    isAndroidTimer() {
      this.timer = setInterval(() => {
        console.log('timer', 'Android')
        this.capture()
      }, 300000)
    },

    isIosTimer() {
      this.timer = setInterval(() => {
        console.log('timer', 'ios')
        this.showUploader = true
      }, 300000)
    },

    async capture() {
      // 指定图片的DataURL(图片的base64编码数据)
      this.context.drawImage(this.video, 0, 0, 150, 230)
      const file = {}
      file.content = this.canvas.toDataURL('image/png', 1)
      file.name = 'face.png'
      file.type = 'image/png'
      const fileImg = await this.dataURLtoFile(file)
      this.uploadImg(fileImg)
    },

    dataURLtoFile(file) {
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
    },

    uploadImg(file) {
      var formData = new FormData()
      formData.append('file', file)
      uploadFile(formData).then(response => {
        this.faceImg = response.data
        this.$toast(this.faceImg)
        if (this.context) {
          this.context.clearRect(0, 0, 150, 230)
        }
        this.sendFaceImg()
      })
    },

    // 人脸照片比对
    sendFaceImg() {
      checkFaceImg({
        faceImg: this.faceImg,
        examResultId: this.submitExamEdit.examResultId,
        userId: this.submitExamEdit.userId
      }).then(response => {
        this.showUploader = false
        if (response.data) {
          this.submitExamEdit.isCheat = 0
          if (this.isSubmit) {
            this.showCanvas = true
          }
        }
      }).catch((e) => {
        this.showUploader = false
        if (e.code < 500) {
          this.submitExamEdit.isCheat = 1
          this.showCanvas = true
        }
      })
    },

    cancleCanvas() {
      this.showCanvas = false
      console.log('取消画布')
    },

    clearCanvas() {
      console.log('清除画布')
    },

    saveCanvas(file) {
      this.$toast.loading()
      var formData = new FormData()
      formData.append('file', file)
      uploadFile(formData).then(response => {
        this.$toast.clear()
        this.$set(this.submitExamEdit, 'signImg', response.data)
        this.showCanvas = false
        this.sendSubmitData()
      }).catch(() => {
        this.$toast.clear()
      })
    },

    async afterReadImg(file) {
      const fileImage = await compressImage(file)
      this.uploadImg(fileImage)
    }
  }
}
</script>
<style lang="scss" scoped>
header {
  width: calc(100% - 30px);
  position: fixed;
  height: 68px;
  left: 0;
  top: 0;
  background: rgba(245, 249, 254, 1);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 15px;
  & > div {
    display: flex;
    align-items: center;
  }
}

.header-left {
  .kuai {
    width: 6px;
    height: 18px;
    background: rgba(78, 140, 238, 1);
    margin-right: 10px;
  }
  .examType {
    font-size: 18px;
    font-weight: 500;
    color: rgba(68, 68, 68, 1);
  }
}
.timeDesc {
  font-size: 12px;
  color: rgba(68, 68, 68, 1);
  margin-right: 5px;
}
.van-count-down-item {
  width: 20px;
  height: 20px;
  background: rgba(57, 57, 57, 1);
  border-radius: 2px;
  display: inline-block;
  margin-right: 2px;
  font-size: 12px;
  text-align: center;
  color: #fff;
}
.header-right {
  .van-icon {
    font-size: 18px;
  }
  .qProgress {
    font-size: 14px;
    margin-left: 5px;
  }
}
section.container {
  margin-bottom: 50px;
}
.qTitle {
  margin-top: 20px;
  display: flex;
  max-height: 330px;
  span {
    font-size: 18px;
    font-weight: 500;
    color: rgba(68, 68, 68, 1);
    line-height: 25px;
    margin-right: 150px;
  }
  video {
    width: 150px;
    height: 230px;
    position: fixed;
    right: 0;
    top: 70px;
  }
  canvas {
    position: fixed;
    right: 0;
    top: 70px;
    width: 150px;
    height: 230px;
  }
}
.selectArea {
  display: flex;
  margin: 20px auto;

}
.attach {
  video {
    width: 330px;
    height: 178px;
    background: rgba(216, 216, 216, 1);
    border-radius: 17px;
  }
  img {
    width: 330px;
    height: 178px;
    background: rgba(216, 216, 216, 1);
    border-radius: 17px;
  }
}
footer {
  display: flex;
  height: 50px;
  align-items: center;
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  justify-content: space-between;
}
.footerLeft {
  width: 244px;
  display: flex;
  align-items: center;
  .qButton {
    flex: 1;
    text-align: center;
    height: 50px;
    line-height: 50px;
    font-size: 14px;
    background: rgba(245, 249, 254, 1);
  }
  .qButton + .qButton {
    border-left: 1px solid #4e8cee;
  }
}

.wrapper{
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
}
</style>
<style lang="scss">
.examIngPage {
  .van-checkbox,
  .van-radio {
    min-height: 40px;
    display: flex;
    align-items: center;
    margin-bottom: 5px;
  }
  .van-checkbox__label,
  .van-radio__label {
    font-size: 16px;
    line-height: 30px;
  }
  .submit {
    flex: 1;
  }
}
</style>
