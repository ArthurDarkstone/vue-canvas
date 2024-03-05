<template>
  <div class="canvas-container">
    <div class="filter-container list-align">

      <label class="label-item">单位大小 : </label>

      <input v-model="box" type="number" class="input-item" @change="handleChangeBox(box)" />
      <button type="primary" icon="el-icon-refresh-right" @click="handleChangeBox(box)"> {{ boxNumber ? '重置画板' : '确 认' }} </button>

      <span class="ctrl-text"> 当前模式: {{ mode ? '固定选择' : '自由选择' }} </span>

      <span class="ctrl-btns">
        <button type="primary" icon="el-icon-refresh" @click="handeleChangeMode()"> 切换模式 </button>
        <button type="primary" icon="el-icon-refresh-left" :disabled="tempStep.length === 1" @click="handlePre()"> 撤 销 </button>
        <button type="primary" icon="el-icon-refresh-right" :disabled="nextStep.length === 0" @click="handleNext()"> 恢 复 </button>
      </span>

    </div>
    <div class="canvas-content" @mousemove="beginPath($event)">
      <canvas
        id="canvas"
        class="canvas-item"
        :width="canvasWidth"
        :height="canvasHeight"
        @mousedown="canvasDown($event)"
        @mouseup="canvasUp($event)"
        @mousemove="canvasMove($event)"
      />

    </div>

  </div>
</template>

<script>
export default {
  name: 'Canvas',
  data() {
    return {
      context: {},
      area: [],
      canvasWidth: 1920,
      canvasHeight: 1080,
      preview: '',
      moving: false,
      preStep: [],
      nextStep: [],
      nextDatas: [],
      tempStep: [],
      tempImgs: [],
      rectPos: {
        x: 0,
        y: 0
      },
      box: 20,
      boxSize: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13],
      moveStep: 0,
      moveStepx: 0,
      moveStepy: 0,
      tempImgData: null,
      boxNumber: 0,
      imgData: '',
      positions: [],
      dialogPicVisable: false,
      tempData: {
        type: null,
        points: [],
        icon: ''
      },
      curImg: null,
      mode: true, // true 为固定 false 为自由
      movePos: {
        x: 1,
        y: 1
      }
    }
  },
  computed: {
    boxWidth() { // 每个盒子的单位长宽
      return this.canvasWidth / this.box
    },
    boxHeight() {
      return this.canvasHeight / this.box
    },
    originPosX() { // 选中盒子的初始位置
      return Math.floor(this.rectPos.x / this.boxWidth)
    },
    originPosY() {
      return Math.floor(this.rectPos.y / this.boxHeight)
    },
    menuWidth() { // 菜单定位适配
      return 0
    },
    device() {
      return 'desktop'
    }
  },
  watch: {
    moveStep(newV, oldV) { // 监听mouseMove 移动时的单位改变量
      const { right, bottom } = this.getShortDistance(this.originPosX, this.originPosY)
      // const topRight = top >= right ? right : top
      const rightBottom = right >= bottom ? bottom : right
      // const bottomLeft = bottom >= left ? left : bottom
      // const leftTop = right >= top ? top : right

      if (this.moveStep > 0) {
        if (this.moveStep > rightBottom) {
          if (this.mode) {
            this.moveStep = rightBottom
            // this.$message.warning('sdddd')
          }
        }
      } else {
        this.moveStep = 0
      }

      if (newV - oldV) { // mouseOver rect选择效果
        this.context.putImageData(this.tempImgData, 0, 0)
      }
    },
    moveStepx(newV, oldV) {
      // if (this.moveStepx > 0) {

      // } else {
      //   this.moveStepx = 0
      //   this.moveStepy = 0
      // }
      if (newV - oldV) { // mouseOver rect选择效果
        this.context.putImageData(this.tempImgData, 0, 0)
      }
    },
    moveStepy(newV, oldV) {
      // if (this.moveStepy > 0) {

      // } else {
      //   this.moveStepx = 0
      //   this.moveStepy = 0
      // }
      if (newV - oldV) { // mouseOver rect选择效果
        this.context.putImageData(this.tempImgData, 0, 0)
      }
    }
  },
  mounted() {
    const canvas = document.querySelector('#canvas')

    this.context = canvas.getContext('2d')
    this.initCanvas()
  },
  methods: {
    initCanvas() { // 初始化
      this.handleChangeBox(this.box)

      // this.tempStep.push(preData)

      this.context.lineWidth = 1
      this.context.shadowBlur = 1
      this.context.shadowColor = '#2c3e50'
      this.context.strokeStyle = '#2c3e50'

      const canvas = document.querySelector('#canvas')
      this.imgData = canvas.toDataURL('image/png')
    },
    resetCanvas() { // 清空画布
      this.context.fillStyle = '#fff'
      this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      this.context.fillRect(0, 0, this.canvasWidth, this.canvasHeight)
      this.context.fillStyle = '#000'
    },
    handleChangeBox(box) { // 盒子单位长度变化
      this.resetCanvas()
      // this.box = box

      // this.moveStep = 0
      this.positions = []
      this.tempStep = []
      this.nextStep = []
      this.nextDatas = []
      this.boxNumber = 0
      this.context.fillStyle = '#fff'

      this.context.strokeRect(0, 0, this.canvasWidth, this.canvasHeight)
      this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      this.context.fillRect(0, 0, this.canvasWidth, this.canvasHeight)

      this.context.lineWidth = 1
      this.context.fillStyle = '#000'
      this.context.shadowBlur = 1
      this.context.shadowColor = '#2c3e50'
      this.context.strokeStyle = '#2c3e50'

      this.resetCanvas()

      for (let i = 0; i < box; i++) { // 绘制虚线
        this.context.font = '12px STheiti, SimHei'
        this.context.fillText(`${i + 1}`, this.boxWidth * (i + 1) - this.boxWidth / 2, 12)
        this.context.fillText(`${i + 1}`, 4, this.boxHeight * (i + 1) - this.boxHeight / 2)
        this.context.beginPath()

        this.context.lineWidth = 1
        this.context.shadowBlur = 0
        this.context.strokeStyle = '#ddd'
        this.context.setLineDash([8, 3, 1, 3])

        this.context.moveTo(this.boxWidth * (i + 1), 0)
        this.context.lineTo(this.boxWidth * (i + 1), this.canvasHeight)
        this.context.stroke()

        this.context.moveTo(0, this.boxHeight * (i + 1))
        this.context.lineTo(this.canvasWidth, this.boxHeight * (i + 1))
        this.context.stroke()

        this.context.closePath()
      }

      this.context.shadowBlur = 1
      this.context.closePath()

      this.handleReset()

      this.context.setLineDash([])

      const curData = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight)
      this.tempStep.push(curData) // tempStep 记录图像，tempStep[0]为初始图像
    },
    handleReset() { // 清空选择的区域为false
      this.area = []

      for (let i = 0; i < this.box; i++) {
        const item = []
        for (let j = 0; j < this.box; j++) {
          item.push(false)
        }
        this.area.push(item)
      }
    },
    beginPath({ target }) { // beginPath
      const canvas = document.querySelector('#canvas')
      if (target !== canvas) {
        this.context.beginPath()
      }
    },
    canvasOver(e) {
      const { target } = e

      // 获取鼠标进入位置
      const canvasX = e.clientX - target.parentNode.offsetLeft - this.menuWidth
      const canvasY = e.clientY - target.parentNode.offsetTop - 50

      this.context.fillText(`${canvasX / this.canvasWidth}`, canvasX, canvasY - this.boxHeight / 2)
    },
    canvasDown(e) {
      if (this.boxNumber > this.box * this.box) { // 根据boxNumber判断是否完成绘制
        this.moving = false
        // this.$message.warning('已完成绘制')

        return
      }

      // reset
      this.moving = true
      this.moveStep = 0
      this.boxNumber++

      const { target } = e
      let canvasOffsetTop
      if (this.device === 'mobile') {
        canvasOffsetTop = 0
      } else {
        canvasOffsetTop = 0
      }
      const canvasOffsetLeft = 0
      const { scrollTop, scrollLeft } = document.documentElement

      // 获取鼠标初始位置
      const canvasX = e.clientX - target.parentNode.offsetLeft - canvasOffsetLeft + scrollLeft
      const canvasY = e.clientY - target.parentNode.offsetTop - canvasOffsetTop + scrollTop
      // 初始位置赋值给全局
      this.rectPos.x = canvasX
      this.rectPos.y = canvasY

      if (this.isSelect(this.originPosX, this.originPosY)) { // 判断区域是否选择
        this.$message.warning('区域已被选中')
        this.boxNumber--

        this.moving = false
      }

      this.context.beginPath()

      this.context.moveTo(canvasX, canvasY)

      const preData = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight)
      this.preStep.push(preData)

      this.tempImgData = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight) // moveEvent
    },
    canvasMove(e) {
      if (this.moving) {
        this.nextStep = []
        this.nextDatas = []

        const { target } = e
        let canvasOffsetTop
        if (this.device === 'mobile') {
          canvasOffsetTop = 0
        } else {
          canvasOffsetTop = 0
        }
        const canvasOffsetLeft = 0

        // 获取鼠标位置
        const canvasX = e.clientX - target.parentNode.offsetLeft - canvasOffsetLeft
        const canvasY = e.clientY - target.parentNode.offsetTop - canvasOffsetTop
        // 获取初始位置
        const { x, y } = this.rectPos

        let nx = Math.ceil((canvasX - x) / this.boxWidth) // 移动的单位距离
        let ny = Math.ceil((canvasY - y) / this.boxHeight) // 移动的单位距离

        const n = nx >= ny ? nx : ny

        this.moveStep = n

        if (n) { // 结合watch 的mouseMove效果
          this.context.fillStyle = '#2883dd'

          if (this.mode) {
            this.context.strokeRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, n * this.boxWidth, (n * this.boxWidth) * 9 / 16)
            this.context.clearRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, n * this.boxWidth, (n * this.boxWidth) * 9 / 16)
            this.context.fillStyle = '#000'

            const textOffset = String(this.boxNumber).length * this.boxHeight / 4

            this.context.font = `${this.boxHeight}px STheiti, SimHei`
            const textAlignX = (this.originPosX + n / 2) * this.boxWidth - textOffset
            const textAlignY = (this.originPosY + n / 2) * this.boxHeight + this.boxHeight / 3
            this.context.fillText(this.boxNumber, textAlignX, textAlignY)

            // w * h
            this.context.font = '18px STheiti, SimHei'
            this.context.fillStyle = '#000'
            this.context.fillRect((this.originPosX + n) * this.boxWidth - 76 - (String(n).length - 1) * 18, (this.originPosY + n) * this.boxHeight - 33, 76 + (String(n).length - 1) * 18, 33)
            this.context.fillStyle = '#fff'
            this.context.fillText(`${n} x ${n}`, (this.originPosX + n) * this.boxWidth - 70 - (String(n).length - 1) * 18 + 10, (this.originPosY + n) * this.boxHeight - 33 + 24)

            // ori
            this.context.font = '18px STheiti, SimHei'
            this.context.fillStyle = '#000'
            this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, 70 + (String(this.originPosX).length + String(this.originPosY).length - 2) * 9, 33)
            this.context.fillStyle = '#fff'
            this.context.fillText(`( ${this.originPosX + 1}, ${this.originPosY + 1} )`, this.originPosX * this.boxWidth, this.originPosY * this.boxHeight + 24)
          } else {
            nx = Math.ceil((canvasX - x) / this.boxWidth)
            ny = Math.ceil((canvasY - y) / this.boxHeight)

            this.moveStepx = nx
            this.moveStepy = ny

            if (this.moveStepx > 0 && this.moveStepy > 0) {
              this.context.strokeRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, nx * this.boxWidth, (ny * this.boxWidth) * 9 / 16)
              this.context.clearRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, nx * this.boxWidth, (ny * this.boxWidth) * 9 / 16)
              this.context.fillStyle = '#000'

              const textOffset = String(this.boxNumber).length * this.boxHeight / 4

              this.context.font = `${this.boxHeight}px STheiti, SimHei`
              const textAlignX = (this.originPosX + nx / 2) * this.boxWidth - textOffset
              const textAlignY = (this.originPosY + ny / 2) * this.boxHeight + this.boxHeight / 3
              this.context.fillText(this.boxNumber, textAlignX, textAlignY)

              // w * h
              this.context.font = '18px STheiti, SimHei'
              this.context.fillStyle = '#000'
              this.context.fillRect((this.originPosX + nx) * this.boxWidth - 76 - (String(nx).length + String(ny).length - 2) * 9, (this.originPosY + ny) * this.boxHeight - 33, 76 + (String(nx).length + String(ny).length - 2) * 9, 33)
              this.context.fillStyle = '#fff'
              this.context.fillText(`${nx} x ${ny}`, (this.originPosX + nx) * this.boxWidth - 70 - (String(nx).length + String(ny).length - 2) * 9 + 10, (this.originPosY + ny) * this.boxHeight - 33 + 24)

              // ori
              this.context.font = '18px STheiti, SimHei'
              this.context.fillStyle = '#000'
              this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, 70 + (String(this.originPosX).length + String(this.originPosY).length - 2) * 9, 33)
              this.context.fillStyle = '#fff'
              this.context.fillText(`( ${this.originPosX + 1}, ${this.originPosY + 1} )`, this.originPosX * this.boxWidth, this.originPosY * this.boxHeight + 24)
            }
          }
        }


      }
    },
    canvasUp(e) {
      if (this.boxNumber > this.box * this.box) { // 判断区域
        this.moving = false
        this.$message.warning('已完成绘制')

        return
      }
      const curData = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight)
      const { target } = e
      let canvasOffsetTop
      if (this.device === 'mobile') {
        canvasOffsetTop = 0
      } else {
        canvasOffsetTop = 0
      }
      const canvasOffsetLeft = 0

      // 获取鼠标结束位置
      const canvasX = e.clientX - target.parentNode.offsetLeft - canvasOffsetLeft
      const canvasY = e.clientY - target.parentNode.offsetTop - canvasOffsetTop
      // 获取初始位置
      const { x, y } = this.rectPos

      let nx = Math.ceil((canvasX - x) / this.boxWidth) // 移动的单位距离
      let ny = Math.ceil((canvasY - y) / this.boxHeight) // 移动的单位距离

      const n = nx >= ny ? nx : ny // 16 : 9 、 1 : 1
      this.moveStep = n

      if (n === 0) {
        this.boxNumber--
      } else {
        this.context.fillStyle = '#2883dd'

        if (this.mode) {
          this.context.strokeRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, n * this.boxWidth, (n * this.boxWidth) * 9 / 16)
          this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, n * this.boxWidth, (n * this.boxWidth) * 9 / 16)
          this.context.fillStyle = '#000'

          const textOffset = String(this.boxNumber).length * this.boxHeight / 4

          this.context.font = `${this.boxHeight}px STheiti, SimHei`
          const textAlignX = (this.originPosX + n / 2) * this.boxWidth - textOffset
          const textAlignY = (this.originPosY + n / 2) * this.boxHeight + this.boxHeight / 3
          this.context.fillText(this.boxNumber, textAlignX, textAlignY)

          // w * h
          this.context.font = '18px STheiti, SimHei'
          this.context.fillStyle = '#000'
          this.context.fillRect((this.originPosX + n) * this.boxWidth - 76 - (String(n).length - 1) * 18, (this.originPosY + n) * this.boxHeight - 33, 76 + (String(n).length - 1) * 18, 33)
          this.context.fillStyle = '#fff'
          this.context.fillText(`${n} x ${n}`, (this.originPosX + n) * this.boxWidth - 70 - (String(n).length - 1) * 18 + 10, (this.originPosY + n) * this.boxHeight - 33 + 24)

          // ori
          // this.context.font = '18px STheiti, SimHei'
          // this.context.fillStyle = '#000'
          // this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, 70 + (String(this.originPosX).length + String(this.originPosY).length - 2) * 9, 33)
          // this.context.fillStyle = '#fff'
          // this.context.fillText(`( ${this.originPosX + 1}, ${this.originPosY + 1} )`, this.originPosX * this.boxWidth, this.originPosY * this.boxHeight + 24)
        } else {
          nx = Math.ceil((canvasX - x) / this.boxWidth)
          ny = Math.ceil((canvasY - y) / this.boxHeight)

          this.moveStepx = nx
          this.moveStepy = ny

          if (this.moveStepx > 0 && this.moveStepy > 0) {
            this.context.strokeRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, nx * this.boxWidth, (ny * this.boxWidth) * 9 / 16)
            this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, nx * this.boxWidth, (ny * this.boxWidth) * 9 / 16)
            this.context.fillStyle = '#000'

            const textOffset = String(this.boxNumber).length * this.boxHeight / 4

            this.context.font = `${this.boxHeight}px STheiti, SimHei`
            const textAlignX = (this.originPosX + nx / 2) * this.boxWidth - textOffset
            const textAlignY = (this.originPosY + ny / 2) * this.boxHeight + this.boxHeight / 3
            this.context.fillText(this.boxNumber, textAlignX, textAlignY)

            // w * h
            this.context.font = '18px STheiti, SimHei'
            this.context.fillStyle = '#000'
            this.context.fillRect((this.originPosX + nx) * this.boxWidth - 76 - (String(nx).length + String(ny).length - 2) * 9, (this.originPosY + ny) * this.boxHeight - 33, 76 + (String(nx).length + String(ny).length - 2) * 9, 33)
            this.context.fillStyle = '#fff'
            this.context.fillText(`${nx} x ${ny}`, (this.originPosX + nx) * this.boxWidth - 70 - (String(nx).length + String(ny).length - 2) * 9 + 10, (this.originPosY + ny) * this.boxHeight - 33 + 24)

            // ori
            // this.context.font = '18px STheiti, SimHei'
            // this.context.fillStyle = '#000'
            // this.context.fillRect(this.originPosX * this.boxWidth, this.originPosY * this.boxHeight, 70 + (String(this.originPosX).length + String(this.originPosY).length - 2) * 9, 33)
            // this.context.fillStyle = '#fff'
            // this.context.fillText(`( ${this.originPosX + 1}, ${this.originPosY + 1} )`, this.originPosX * this.boxWidth, this.originPosY * this.boxHeight + 24)
          }
        }

        if (this.moving) {
          // this.handleSelect() // 有撤销暂时注释掉

          const imgData = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight)
          const canvas = document.querySelector('#canvas')
          const idata = canvas.toDataURL('image/png')
          this.imgData = idata

          if (this.mode) {
            if (this.moveStep >= 0) {
              this.positions.push({ // createData 的参数
                x: Math.round(this.originPosX * this.boxWidth * 10) / 10,
                y: Math.round(this.originPosY * this.boxHeight * 10) / 10,
                w: Math.round(this.moveStep * this.boxWidth * 10) / 10,
                h: Math.round(this.moveStep * this.boxHeight * 10 / 10),
                index: this.boxNumber
              })
              console.log(`
                x: ${Math.round(this.originPosX * this.boxWidth * 10) / 10},
                y: ${this.originPosY} * ${this.boxHeight},
                w: ${this.moveStep} * ${this.boxWidth},
                h: ${this.moveStep} * ${this.boxHeight},
                index: ${this.boxNumber}
              `)
            }
          } else {
            if (this.moveStepx > 0 && this.moveStepy > 0) {
              this.positions.push({ // createData 的参数
                x: Math.round(this.originPosX * this.boxWidth * 10) / 10,
                y: Math.round(this.originPosY * this.boxHeight * 10) / 10,
                w: Math.round(this.moveStepx * this.boxWidth * 10) / 10,
                h: Math.round(this.moveStepy * this.boxHeight * 10 / 10),
                index: this.boxNumber
              })
              console.log(`
                x: ${this.originPosX} * ${this.boxWidth},
                y: ${this.originPosY} * ${this.boxHeight},
                w: ${this.moveStepx} * ${this.boxWidth},
                h: ${this.moveStepy} * ${this.boxHeight},
                index: ${this.boxNumber}
              `)
            }
          }
          this.tempStep.push(imgData) // 撤销操作
        } else {
          // this.handleDisSelect(this.originPosX, this.originPosY, this.moveStep)
        }
      }

      this.context.closePath()

      this.moving = false
      this.tempImgs.push(curData)
    },
    handleSelect() {
      for (let i = 0; i < this.box; i++) {
        for (let j = 0; j < this.box; j++) {
          if (i >= this.originPosX && i < this.originPosX + this.moveStep && j >= this.originPosY && j < this.originPosY + this.moveStep) {
            this.area[i][j] = true
          }
        }
      }
    },
    handleDisSelect(x, y, n) {
      for (let i = 0; i < this.box; i++) {
        for (let j = 0; j < this.box; j++) {
          if (i >= this.originPosX && i < this.originPosY + n && j >= this.originPosY && j < this.originPosY + n) {
            this.area[i][j] = false
          }
        }
      }
    },
    isSelect(x, y) {
      if (this.area[x][y]) {
        return true
      } else {
        return false
      }
    },
    getShortDistance(x, y) { // 判断上右下左的单位最短距离
      let [top, right, bottom, left] = [1, 1, 1, 1]

      for (let i = y - 1; i >= 0; i--) {
        if (!this.area[x][i]) {
          top++
        } else {
          break
        }
      }

      for (let i = x + 1; i < this.box; i++) {
        if (!this.area[i][y]) {
          right++
        } else {
          right

          break
        }
      }

      for (let i = y + 1; i < this.box; i++) {
        if (!this.area[x][i]) {
          bottom++
        } else {
          bottom

          break
        }
      }

      for (let i = x - 1; i >= 0; i--) {
        if (!this.area[i][y]) {
          left++
        } else {
          break
        }
      }

      return { top, right, bottom, left }
    },
    handlePostData() {
      this.dialogPicVisable = true

      const postData = {
        type: this.boxNumber,
        points: JSON.stringify(this.positions),
        icon: this.imgData
      }

      this.tempData = { ...postData }
    },
    async postPicData() {
      try {
        const msg = '成功'
        // const { msg } = await createLayout({ ...this.tempData })

        if (msg === '成功') {
          this.$message.success('上传成功')
          this.handleChangeBox(this.box)
          this.dialogPicVisable = false
          this.$router.push({
            path: '/video/index'
          })
        }
      } catch (error) {
        this.$message.error('添加失败')
      }
    },
    handlePre() { // 撤销操作
      if (this.boxNumber > 0 && this.boxNumber < this.box * this.box) {
        const imgData = this.tempStep.pop()
        this.nextStep.unshift(imgData)

        const curPos = this.positions.pop()
        this.nextDatas.unshift(curPos)

        this.boxNumber--

        this.handleReset()
        this.context.putImageData(this.tempStep[this.boxNumber], 0, 0)

        const canvas = document.querySelector('#canvas')
        const dataUrl = canvas.toDataURL('image/png')

        this.curImg = dataUrl
      }
    },
    handleNext() { // 恢复撤销操作
      if (this.nextStep.length > 0 && this.boxNumber < this.box * this.box) {
        const imgData = this.nextStep.shift()
        const curData = this.nextDatas.shift()

        this.tempStep.push(imgData)
        this.positions.push(curData)

        this.context.putImageData(imgData, 0, 0)
        this.boxNumber++

        this.handleReset()

        const canvas = document.querySelector('#canvas')
        const dataUrl = canvas.toDataURL('image/png')

        this.curImg = dataUrl
      }
    },
    handeleChangeMode() {
      this.mode = !this.mode
      // this.handleChangeBox(this.box)
    }
  }
}
</script>

<style lang="scss" scoped>
.canvas-container {
  padding: 20px;

  .canvas-content {
    width: 1920px;
    height: 1080px;
    margin-top: 8px;

    .canvas-item {
      border: 1px solid #2c3e50;
    }
  }
}
.submit-container {
  margin: 25px;
}
.list-align {
  width: 1920px;

  .ctrl-btns {
    margin-left: 45px;
  }

  .ctrl-text {
    margin-left: 25px;
  }
}
.label-item {
  font-weight: 100;
  font-size: 20px;
  height: 40px;
  line-height: 40px;
}
.input-item {
  width: 120px;
  margin-right: 25px;
}
.placeholder {
  height: 1000px;
}
</style>
