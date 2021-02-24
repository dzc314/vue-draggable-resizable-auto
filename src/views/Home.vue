<template>
  <div class="home" ref="page">
    <vue-draggable-resizable
      v-for="element in myArray"
      :key="element.id"
      @dragging="onDragging"
      @dragstop="onDragstop"
      @resizing="onResizing"
      @mousedown.native="mousedown(element)"
      :parent="true"
      :w="element.width"
      :h="element.height"
      :x="element.x"
      :y="element.y"
      :min-width="120"
      :min-height="120"
      :grid="[40, 40]"
      :style="`transform: translate(${element.x}px,${element.y}px)`"
    >
      <p>
        {{ element.name }} <br />
        X: {{ element.x }} <br />
        Y: {{ element.y }} <br />
        Width:{{ element.width }} <br />
        Height:{{ element.height }}
      </p>
    </vue-draggable-resizable>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { find, sortBy, throttle } from 'lodash'
interface PageElement {
  id?: number
  name?: string
  x: number
  y: number
  width: number
  height: number
}
@Component({
  components: {},
})
export default class Home extends Vue {
  pageWidth = 1920
  myArray: PageElement[] = []
  editElement: PageElement = {
    x: 0,
    y: 0,
    width: 0,
    height: 0,
  }
  created() {
    this.myArray = new Array(6).fill('').map((item, index) => {
      const id = index + 1
      const element: PageElement = {
        id,
        name: 'test' + id,
        x: Math.round(Math.random() * 10) * 40,
        y: Math.round(Math.random() * 10) * 40,
        width: 160,
        height: 160,
      }
      return element
    })
    // console.log(this.myArray)
  }
  mounted() {
    // console.log(this.$refs.page)

    this.pageWidth = (this.$refs.page as any).offsetWidth
    this.updateElementPosition()
  }
  updateElementPosition = throttle(this.updateElementPositionFn, 100)
  /*
   * 更新所有元素定位
   */
  updateElementPositionFn() {
    let fillArr: PageElement[] = []
    // 所有元素按y坐标升序排序
    this.myArray = sortBy(this.myArray, ['y', 'x'])
    console.table(this.myArray.map((item) => item.name))
    // console.log(this.pageWidth)

    this.myArray.forEach((ele: PageElement, index: number) => {
      // console.log(ele)
      // 更新填充数据
      function updateFillArr(ele: PageElement) {
        const sameRowEle = find(fillArr, (fillEle: PageElement) => {
          return (
            (fillEle.width + fillEle.x === ele.x ||
              ele.x + ele.width === fillEle.x) &&
            fillEle.y === ele.y &&
            fillEle.height === ele.height
          )
        })
        // console.log(sameRowEle)
        if (sameRowEle) {
          // 合并同行等高填充元素
          // 插入元素在左侧
          if (ele.x < sameRowEle.x) {
            sameRowEle.x -= ele.width
          }
          sameRowEle.width += ele.width
          // todo 是否需插入一个空元素
        } else {
          fillArr.push({
            name: ele.name,
            x: ele.x,
            y: ele.y,
            width: ele.width,
            height: ele.height,
          })
        }
        fillArr = sortBy(fillArr, ['y', 'x'])
      }

      // 找到当前元素的上一行阻挡元素
      function findTopY(ele: PageElement) {
        const len = fillArr.length
        let y = 0
        // 从下往上找
        for (let i = len; i >= 0; i--) {
          const fillEle = fillArr[i]
          // console.log(fillEle)
          if (!fillEle) {
            continue
          }
          if (
            (fillEle.x <= ele.x && fillEle.x + fillEle.width > ele.x) ||
            (ele.x + ele.width > fillEle.x &&
              ele.x + ele.width <= fillEle.x + fillEle.width)
          ) {
            y = Math.max(fillEle.y + fillEle.height, y)
          }
        }
        return y
      }
      // 第一个元素优先处理
      if (index === 0) {
        ele.y = 0
        updateFillArr(ele)
        return
      }
      // 找到拦截的y点
      const y = findTopY(ele)
      if (y) {
        ele.y = y
        updateFillArr(ele)
        return
      }

      for (let i = 0; i < fillArr.length; i++) {
        const fillEle = fillArr[i]
        if (!fillEle) {
          continue
        }

        let spaceStar = 0
        spaceStar = fillEle.x + fillEle.width
        let spaceEnd = this.pageWidth
        const nextEle = fillArr[i + 1]
        // todo nextEle.y === fillEle.y逻辑要改为是否在同一行的判断
        // 从右侧插入
        if (nextEle && nextEle.y < fillEle.y + fillEle.height) {
          spaceEnd -= spaceEnd - nextEle.x
        }
        if (spaceStar <= ele.x && spaceEnd >= ele.x + ele.width) {
          ele.y = fillEle.y
          console.log(fillEle)

          console.log(ele)

          updateFillArr(ele)
          // ele.x = fillEle.x + fillEle.width
          return
        }
        // 从左侧插入
        spaceEnd = fillEle.x
        const prevEle = fillArr[i - 1]
        let leftSpaceX = 0
        if (i != 0 && prevEle && prevEle.y >= fillEle.y) {
          leftSpaceX = prevEle.x + prevEle.width
          spaceEnd -= leftSpaceX
        }
        if (ele.x + ele.width <= spaceEnd && ele.x >= leftSpaceX) {
          // console.log(fillEle)
          ele.y = fillEle.y
          updateFillArr(ele)
          return
        }
        ele.y = fillEle.y + fillEle.height
      }
      // console.log(fillArr)
      updateFillArr(ele)
    })
  }
  // 触发选中操作元素
  mousedown(ele: PageElement) {
    // console.log(ele)
    this.editElement = ele
  }
  // 拖拽操作
  onDragging(x: number, y: number) {
    // console.log(x, y)
    this.editElement.x = x
    this.editElement.y = y
    // this.updateElementPosition()
  }
  onDragstop() {
    // console.log(x, y)
    // this.editElement.x = x
    // this.editElement.y = y
    this.updateElementPosition()
  }

  // 元素大小操作
  onResizing(x: number, y: number, width: number, height: number) {
    // console.log(width, height)
    this.editElement.x = x
    this.editElement.y = y
    this.editElement.width = width
    this.editElement.height = height
    this.updateElementPosition()
  }
}
</script>
<style lang="scss" scoped>
.home {
  position: relative;
  width: 1000px;
  height: 1000px;
  > div {
    border: 1px solid #ddd;
    cursor: pointer;
  }
}
</style>
