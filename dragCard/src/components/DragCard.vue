<template>
  <div class="dragCard">
    <div
      class="dragCard_warpper"
      ref="dragCard_warpper"
      :style="dragCardWarpperStyle">
      <div
        v-for="(item, index) in list"
        :key="index"
        class="dragCard_item"
        :style="initItemStyle(index)"
        :ref="item.dragCard_id">
        <div class="dragCard_content">
          <div
            class="dragCard_head"
            @mousedown="handleMousedown($event, item)">
            <slot name="head" :item="item" >
              <div class="dragCard_head-defaut">
                {{item.head ? item.head : `卡片标题${index + 1}`}}
              </div>
            </slot>
          </div>
          <div class="dragCard_body">
            <slot name="content" :item="item">
              <div class="dragCard_body-defaut">
                {{item.content ? item.content : `暂无数据`}}
              </div>
            </slot>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  export default {
    name: 'DragCard',
    props: {
      list: {
        type: Array,
        default: () => {
          return []
        }
      },
      col: {
        type: Number,
        default: 3
      },
      itemWidth: {
        type: Number,
        default: 150
      },
      itemHeight: {
        type: Number,
        default: 150
      }
    },
    data() {
      return {
        timer: null,
        row: 0,
        dragCardWarpperStyle: ''
      };
    },
    created() {

    },
    watch: {
      list: {
        handler: function(newVal, oldVal) {
          this.init();
        },
        immediate: true // 定义的时候就执行一次
      }
    },
    methods: {
      init () {
        // 根据数组的长度length和每行个数col，可以计算出需要多少行row，超出不满一行算一行，用ceil向上取整；
        this.row = Math.ceil(this.list.length / this.col);
        // 计算出容器的宽高
        this.dragCardWarpperStyle = `width: ${this.col * this.itemWidth}px; height:${this.row * this.itemHeight}px`;
        /*
        * 这里处理下数组，引入两个重要的属性：
        * dragCard_id：
        *   给每一个卡片创建一个唯一id，作为ref值，后续通过this.$refs[dragCard_id]获取卡片的dom
        * dragCard_index：
        *   这是每个卡片的位置序号，用于记录卡片当前位置
        * */
        this.list.forEach((item, index) => {
          this.$set(item, 'dragCard_index', index);
          this.$set(item, 'dragCard_id', 'dragCard_id' + index);
        });
      },
      initItemStyle(INDEX) {
        return {
          width: this.itemWidth + 'px',
          height: this.itemHeight + 'px',
          left: (INDEX < this.col ? INDEX : (INDEX % this.col)) * this.itemWidth + 'px',
          top: Math.floor(INDEX / this.col) * this.itemHeight + 'px'
        };
      },
      handleMousedown(e, optionItem) {
        e.preventDefault();
        let that = this;
        if (this.timer) return false;

        // 拷贝一份list，并加上后续要使用的属性；
        let itemList = that.list.map(item => {
          // 如果ref是动态赋的值，存入$refs中会是一个数组；
          let dom = this.$refs[item.dragCard_id][0];
          let left = parseInt(dom.style.left.slice(0, dom.style.left.length - 2));
          let top = parseInt(dom.style.top.slice(0, dom.style.top.length - 2));
          let isMoveing = false; // 标记正在移动的卡片，正在移动的卡片不参与碰撞检测
          return {...item, dom, left, top, isMoveing};
        });

        // 当前卡片对象用的比较多，用一个别名curItem把他存起来；
        let curItem = itemList.find(item => item.dragCard_id === optionItem.dragCard_id);
        curItem.dom.style.transition = 'none';
        curItem.dom.style.zIndex = '100';
        curItem.dom.childNodes[0].style.boxShadow = '0 0 5px rgba(0, 0, 0, 0.1)';
        curItem.startLeft = curItem.left; // 起始的left
        curItem.startTop = curItem.top; // 起始的top
        curItem.OffsetLeft = 0; // left的偏移量
        curItem.OffsetTop = 0; // top的偏移量

        // 即将交换位置的对象
        let targetItem = null;

        // 记录鼠标起始位置
        let mousePosition = {
          startX: e.screenX,
          startY: e.screenY
        };

        document.addEventListener("mousemove", handleMousemove);
        document.addEventListener("mouseup", handleMouseUp);

        // 节流定时器
        let DectetTimer = null;

        // 鼠标移动
        function handleMousemove(e) {
          curItem.OffsetLeft = parseInt(e.screenX - mousePosition.startX);
          curItem.OffsetTop = parseInt(e.screenY - mousePosition.startY);
          curItem.dom.style.left = curItem.startLeft + curItem.OffsetLeft + 'px';
          curItem.dom.style.top = curItem.startTop + curItem.OffsetTop + 'px';
          // 碰撞检测，做一下节流
          if (!DectetTimer) {
            DectetTimer = setTimeout(() => {
              cardDetect();
              clearTimeout(DectetTimer);
              DectetTimer = null;
            }, 200)
          }
        }

        // 卡片移动检测
        function cardDetect() {
          // 根据移动的距离计算出移动到哪一个位置
          let colNum = Math.round((curItem.OffsetLeft / that.itemWidth));
          let rowNum = Math.round((curItem.OffsetTop / that.itemHeight));
          // 这里的dragCard_index需要用到最初点击卡片的位置，因为curItem在后续的卡片交换中dragCard_index已经改变；
          let targetItemDragCardIndex = optionItem.dragCard_index + colNum + (rowNum * that.col);

          // 超出行列，目标位置不变或不存在都直接return；
          if(Math.abs(colNum) >= that.col
            || Math.abs(rowNum) >= that.row
            || Math.abs(colNum) >= that.col
            || Math.abs(rowNum) >= that.row
            || targetItemDragCardIndex === curItem.dragCard_index
            || targetItemDragCardIndex < 0
            || targetItemDragCardIndex > that.list.length - 1) return false;

          let item = itemList.find(item => item.dragCard_index === targetItemDragCardIndex);
          item.isMoveing = true;
          // 将目标卡片拷贝一份，主要是为了松开鼠标的时候赋值给当前卡片；
          targetItem = {...item};
          swicthPosition();
        }

        /*
        * 卡片交换
        * 卡片交换分为两种情况；
        *   1. 当目标位置比当前移动卡片的原位置大的时候，相隔的卡片和目标卡片都要后移一个位置；
        *   2. 当目标位置比当前移动卡片的原位置小的时候，相隔的卡片和目标卡片都要前移一个位置；
        * 注意：
        *   1. 这里有个注意的点是当我们移动的时候，我们拿的是前一个或者后一个的值，所以我们遍历数组的时候要注意从目标值开始遍历；
        *   2. itemList是list的备份，当我们修改了卡片的dragCard_index之后，需要同步到list中；
        * */
        function swicthPosition() {
          const dragCardIndexList = itemList.map(item => item.dragCard_index);
          // 目标卡片位置大于当前卡片位置；
          if (targetItem.dragCard_index > curItem.dragCard_index) {
            for (let i = targetItem.dragCard_index; i >= curItem.dragCard_index + 1; i--) {
              let item = itemList[dragCardIndexList.indexOf(i)];
              let preItem = itemList[dragCardIndexList.indexOf(i - 1)];
              item.isMoveing = true;
              item.left = preItem.left;
              item.top = preItem.top;
              item.dom.style.left = item.left + 'px';
              item.dom.style.top = item.top + 'px';
              item.dragCard_index = that.list[dragCardIndexList.indexOf(i)].dragCard_index -= 1;
              setTimeout(() => {
                item.isMoveing = false;
              }, 300)
            }
          }
          // 目标卡片位置小于当前卡片位置；
          if (targetItem.dragCard_index < curItem.dragCard_index) {
            for (let i = targetItem.dragCard_index; i <= curItem.dragCard_index - 1; i++) {
              let item = itemList[dragCardIndexList.indexOf(i)];
              let nextItem = itemList[dragCardIndexList.indexOf(i + 1)];
              item.isMoveing = true;
              item.left = nextItem.left;
              item.top = nextItem.top;
              item.dom.style.left = item.left + 'px';
              item.dom.style.top = item.top + 'px';
              item.dragCard_index = that.list[dragCardIndexList.indexOf(i)].dragCard_index += 1;
              setTimeout(() => {
                item.isMoveing = false;
              }, 300)
            }
          }
          curItem.left = targetItem.left;
          curItem.top = targetItem.top;
          curItem.dragCard_index =  targetItem.dragCard_index;

          that.$emit('change', itemList.map(item => item.dragCard_index));
        }

        function handleMouseUp() {
          //移除所有监听
          document.removeEventListener("mousemove", handleMousemove);
          document.removeEventListener("mouseup", handleMouseUp);

          // 清除检测的定时器并做最后一次碰撞检测
          clearTimeout(DectetTimer);
          DectetTimer = null;
          cardDetect();

          curItem.dom.style.transition = '';
          that.list.find(item => item.dragCard_id === optionItem.dragCard_id).dragCard_index = curItem.dragCard_index;
          that.$emit('mouseUp', that.list.map(item => item.dragCard_index));
          curItem.dom.style.left = curItem.left + 'px';
          curItem.dom.style.top = curItem.top + 'px';
          that.timer = setTimeout(() => {
            curItem.dom.style.zIndex = '';
            curItem.dom.childNodes[0].style.boxShadow = 'none';
            clearTimeout(that.timer);
            that.timer = null;
          }, 300);
        }
      }
    }
  };
</script>

<style scoped lang="scss">
  .dragCard {
    text-align: center;
    margin: 0 auto;
    &_warpper {
      position: relative;
      margin: 0 auto;
    }

    &_item {
      position: absolute;
      z-index: 0;
      box-sizing: border-box;
      transition: all 300ms;
      padding: 0 5px 10px;
      & > div {
        height: 100%;
      }
    }

    &_content {
      background: #fff;
      box-sizing: border-box;
      border: 1px solid #ccc;
      border-radius: 5px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      overflow: hidden;
    }
    &_head {
      box-sizing: border-box;
      cursor: grab;
      outline: none;
      color: #222;
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: flex-start;
      width: 100%;
      font-size: 14px;
      &-defaut {
        border-bottom: 1px solid #ccc;
        background: #495967;
        color: #fff;
        height: 40px;
        flex-shrink: 0;
        padding: 0 20px;
        width: 100%;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: flex-start;
      }
    }
    &_body {
      box-sizing: border-box;
      height: 100%;
      width: 100%;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-content: center;
      color: #838383;
    }
  }
</style>