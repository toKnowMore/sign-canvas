<!-- 模拟考试 -->
<template>
    <sign-canvas
      :is-cheat="submitExamEdit.isCheat"
      :show="showCanvas"
      @cancle="cancleCanvas"
      @save="saveCanvas"
      @clear="clearCanvas"/>
  </div>
</template>

<script>
import signCanvas from '@/components/signCanvas'
export default {
  components: {
    signCanvas
  },
  data() {
    return {
      showCanvas: false
    }
  }
  methods: {
    cancleCanvas() {
      this.showCanvas = false
      console.log('取消画布')
    },

    clearCanvas() {
      console.log('清除画布')
    },

    saveCanvas(file) {
      consoe.log(file)
    }
  }
}
</script>
