<template>
  <div ref="page" :style="pageStyle">
    <div class="bi-layout" :style="layoutStyle">
      <vue-draggable-resizable
        v-for="element in myArray"
        :key="element.id"
        @dragging="onDragging"
        @dragstop="onDragstop"
        @resizing="onResizing"
        @mousedown.native="mousedown(element)"
        :w="element.width"
        :h="element.height"
        :x="element.x"
        :y="element.y"
        :min-width="120"
        :min-height="60"
        :grid="[20, 20]"
        :scale="scale"
        :handles="handles"
        drag-handle=".view-box"
        :style="`transform: translate(${element.x}px,${element.y}px);width: ${element.width}px`"
      >
        <div class="view-box">
          {{ element.name }} <br />
          X: {{ element.x }} <br />
          Y: {{ element.y }} <br />
          Width:{{ element.width }} <br />
          Height:{{ element.height }}
        </div>
      </vue-draggable-resizable>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { find, max, sortBy, throttle } from 'lodash'
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
  pageWidth = 1430
  pageHeight = 800
  scale = 1
  handles: string[] = ['tl', 'tr', 'br', 'bl']
  myArray: PageElement[] = []
  editElement: PageElement = {
    x: 0,
    y: 0,
    width: 0,
    height: 0,
  }
  get layoutStyle() {
    return {
      width: `${this.pageWidth}px`,
      height: `${this.pageHeight}px`,
      transform: `scale(${this.scale})`,
    }
  }
  get pageStyle() {
    return {
      height: `${this.pageHeight * this.scale}px`,
    }
  }
  created() {
    this.dataInit()
    // console.log(this.myArray)
  }
  mounted() {
    // console.log(this.$refs.page)
    this.pageInit()
    this.updateElementPosition()
    window.addEventListener('resize', this.pageInit)
  }
  beforeDestroy() {
    window.removeEventListener('resize', this.pageInit)
  }

  pageInit() {
    const pageWidth = (this.$refs.page as any).offsetWidth
    // const pageWidth = document.body.offsetWidth
    this.scale = Math.round((pageWidth / this.pageWidth) * 10000) / 10000
  }
  // 初始化demo数据
  dataInit() {
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
  }

  updateElementPosition = throttle(this.updateElementPositionFn, 100)
  /*
   * 更新所有元素定位
   */
  updateElementPositionFn() {
    let fillArr: PageElement[] = []
    let fillHeight = this.pageHeight
    // 所有元素按y坐标升序排序
    this.myArray = sortBy(this.myArray, ['y', 'x'])
    // console.table(this.myArray.map((item) => item.name))
    this.myArray.forEach((ele: PageElement, index: number) => {
      // 范围限制
      if (ele.x < 0) {
        ele.x = Math.max(0, ele.x)
      } else {
        ele.x = Math.min(this.pageWidth - 10 - ele.width, ele.x)
      }

      // 更新填充数据
      const updateFillArr = (ele: PageElement) => {
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
          if (ele.x < sameRowEle.x) {
            // 插入元素在左侧
            sameRowEle.x -= ele.width
          }
          sameRowEle.width += ele.width
        } else {
          fillArr.push({
            x: ele.x,
            y: ele.y,
            width: ele.width,
            height: ele.height,
          })
        }
        fillArr = sortBy(fillArr, ['y', 'x'])
        fillHeight = Math.max(fillHeight, ele.y + ele.height + 200)
      }

      // 找到当前元素的上一行阻挡元素
      function findTopY(ele: PageElement) {
        const len = fillArr.length - 1
        let y = 0
        // 从下往上找
        for (let i = len; i >= 0; i--) {
          const fillEle = fillArr[i]
          // console.log(fillEle)
          if (!fillEle) {
            continue
          }
          const fillEnd = fillEle.x + fillEle.width
          const eleEnd = ele.x + ele.width
          if (ele.x >= fillEnd || eleEnd <= fillEle.x) {
            continue
          }
          y = Math.max(fillEle.y + fillEle.height, y)
          // if (
          //   (ele.x >= fillEle.x && ele.x < fillEnd) ||
          //   (eleEnd > fillEle.x && eleEnd <= fillEnd) ||
          //   (ele.x <= fillEle.x && eleEnd >= fillEnd)
          // ) {
          //   y = Math.max(fillEle.y + fillEle.height, y)
          // }
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
      ele.y = findTopY(ele)
      updateFillArr(ele)
      return

      // for (let i = 0; i < fillArr.length; i++) {
      //   const fillEle = fillArr[i]
      //   if (!fillEle) {
      //     continue
      //   }

      //   let spaceStar = 0
      //   spaceStar = fillEle.x + fillEle.width
      //   let spaceEnd = this.pageWidth
      //   const nextEle = fillArr[i + 1]
      //   // todo nextEle.y === fillEle.y逻辑要改为是否在同一行的判断
      //   // 从右侧插入
      //   if (nextEle && nextEle.y === fillEle.y) {
      //     spaceEnd -= spaceEnd - nextEle.x
      //   }
      //   if (spaceStar <= ele.x && spaceEnd >= ele.x + ele.width) {
      //     ele.y = fillEle.y
      //     updateFillArr(ele)
      //     return
      //   }
      //   // 从左侧插入
      //   spaceEnd = fillEle.x
      //   const prevEle = fillArr[i - 1]
      //   let leftSpaceX = 0
      //   if (i != 0 && prevEle && prevEle.y >= fillEle.y) {
      //     leftSpaceX = prevEle.x + prevEle.width
      //     spaceEnd -= leftSpaceX
      //   }
      //   if (ele.x + ele.width <= spaceEnd && ele.x >= leftSpaceX) {
      //     ele.y = fillEle.y
      //     updateFillArr(ele)
      //     return
      //   }
      // }
    })
    this.pageHeight = fillHeight
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
    if (x < 0) {
      width += x
    }
    x = Math.max(x, 0)
    this.editElement.x = x
    this.editElement.y = y
    width = Math.min(this.pageWidth - 10 - x, width)
    this.editElement.width = width
    this.editElement.height = height
    this.updateElementPosition()
  }
}
</script>
<style lang="scss" scoped>
* {
  box-sizing: border-box;
}
.bi-layout {
  position: relative;
  width: 100%;
  height: 1000px;
  padding: 5px;
  transform-origin: 0 0;
  background-color: #ebeaef;
  // transform: scale(0.8);
}
/deep/.draggable.resizable {
  border: none;
  padding: 5px;
  cursor: pointer;
  .view-box {
    height: 100%;
    border: 1px solid #ddd;
    padding: 10px;
    background-color: #fff;
    overflow: hidden;
  }
  &.active > .view-box {
    box-shadow: 0 0 7px #ccc;
  }
  > .handle {
    width: 15px;
    height: 15px;
    opacity: 0;
    &.handle-tl {
      top: 0;
      left: 0;
    }
    &.handle-tr {
      top: 0;
      right: 0;
    }
    &.handle-bl {
      bottom: 0;
      left: 0;
    }
    &.handle-br {
      bottom: 0;
      right: 0;
    }
  }
}
</style>
