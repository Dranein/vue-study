# 拖拽排列卡片组件
![预览图](/blog/dragCard.gif)

## 前言
之前在掘金上看到了一遍分享拖拽卡片组件的文章，看了大致思路，觉得很清晰，也想动手实现一下；

在过程中发现了蛮多的细节问题，完成后对比了原作者的代码，发现许多可以优化的地方，在这里记录一下；


以下是个人学习实现的demo和源码地址：
- [在线链接](https://dranein.github.io/components/dragCard/)
- [源码地址](https://github.com/Dranein/vue-study/tree/master/dragCard/src)

## 使用
在仓库中拿到dragCard.vue文件，引入到项目中，来看下面这个例子

```javascript
// app.js
<template>
  <div id="app">
    <DragCard
      :list="list"
      :col="4"
      :itemWidth="150"
      :itemHeight="150"
      @change="handleChange"
      @mouseUp="handleMouseUp">
    </DragCard>
  </div>
</template>

<script>
  import DragCard from './components/DragCard.vue'

  export default {
    name: 'app',
    components: {
      DragCard
    },
    data() {
      return {
        list: [
          {head: '标题0', content: "演示卡片0"},
          {head: '标题1', content: "演示卡片1"},
          {head: '标题2', content: "演示卡片2"}
        ],
      }
    },
    methods: {
      handleChange(data) {
        console.log(data);
      },
      handleMouseUp(data) {
        console.log(data);
      }
    }
  }
</script>
```


### 来看看props和方法
通过组件属性和方法可以快速了解整个组件的使用；

#### 属性
|   属性     |  说明                      |  类型  | 默认值|
|:----------- |:------------------------- | :------| :-----|
| list       | 卡片数据                   | Array  | []   |
| col        | 每一行显示多少张卡片        | Number | 3    |
| itemWidth  | 每个卡片的宽度（包括外边距） | Number | 150 |
| itemHeight | 每个卡片的高度（包括外边距） | Number | 150 |

#### 方法
|   方法   |  说明         |  返回值         |
| :---------|:-------------|:-------------|
| @change  |当卡片位置变动的时候触发 |返回的是数组中每一项的位置序号数组|
| @mouseUp |当拖拽完卡片松手的时候触发 | 同上 |
::: tip
返回值是数组中每一项的位置序号集合；返回值数组`index`和`list`中的`index`一致；后续我们可以通过操作这两个数组，合并成`[{ id: 'cardid1', seatid: '1' }...]`这样的形式传递给后端，修改卡片的位置数据；当然建议是在mouseUp的时候去发送请求更优；
:::

#### 插槽slot
| slotName |   说明       | data |
|:---------|:-------------|:-------------|
| head     |卡片的头部标题部分 | listItem |
| content  |卡片内容部分       | listItem |
::: tip
这两个作用域插槽都有默认值，如果不填写的话，标题将显示`list`中的`head`属性，而内容将显示`content`属性；两个`slot`都带上了当前卡片的`list`项数据；可以更加灵活的自定义卡片内容；
:::


## 具体实现
### 大概思路
- 页面卡片采用`absolute`布局，通过设置`left`和`top`，让卡片按顺序排列，因此传入的`list`必须是正序的；
- 初始化样式，通过`props`传入的值，我们可以计算出行列数，卡片位置等信息；
- 给数组中的每一项添加一个位置标识属性，后续的位置交换都可以通过这个标识标识来展开，也是后面触发方法给父级传递的返回值；
- 当鼠标按下的时候，记录下鼠标的当前位置作为起始位置，，当前卡片作为参数传入，并绑定`mousemove`和`mouseup`事件；这时候鼠标的移动距离就是卡片的移动距离；
- 在卡片移动的时候，我们计算出当前是否移动到其他的卡片位置，是的话，相隔之间的所有卡片向后移或向前移，触发父组件的`change`方法；
- 当鼠标松开的时候，卡片回到目标位置,触发父组件的`mouseUp`方法；


### 首先看下页面结构
``` js {16,17,18,19,20,23,24,25,26,27}
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
            @mousedown="touchStart($event, item)">
            <slot name="head" :item="item" >
              <div class="dragCard_head-defaut">
                {{ item.head ? item.head : `卡片标题${index + 1}` }}
              </div>
            </slot>
          </div>
          <div class="dragCard_body">
            <slot name="content" :item="item">
              <div class="dragCard_body-defaut">
                {{ item.content ? item.content : `暂无数据` }}
              </div>
            </slot>
          </div>
        </div>
      </div>
    </div>
  </div>
```
-  鼠标点击标题可以拖动卡片，所以`@mousedown`设置在`dragCard_head`中，为了实现这一点，把`slot`分为了两个部分，一个是`head`标题部分，默认显示`item.content`；一个是`content`内容部分，默认显示`item.head`，；用户可以通过`slot`自定义卡片；[slot知识点](https://cn.vuejs.org/v2/guide/components-slots.html#%E4%BD%9C%E7%94%A8%E5%9F%9F%E6%8F%92%E6%A7%BD)

``` js {11,12,13,14,15,16}
// app.js  使用自定义卡片样式
<template>
  <div id="app">
    <DragCard
      :list="list"
      :col="4"
      :itemWidth="150"
      :itemHeight="150"
      @change="handleChange"
      @mouseUp="handleMouseUp">
      <template v-slot:head="{ item }">
        <div class="dragHead">{{item.head}}</div>
      </template>
      <template v-slot:content="{ item }">
        <div class="dragContent">{{item.content}}</div>
      </template>
    </DragCard>
  </div>
</template>
```
- dragCardWarpperStyle为容器的样式，通过props传入的值计算出容器的宽高；在组件初始化的时候就应该去计算了，用init()包含起来；

``` js
// ...
 created() {
   this.init();
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
   // 通过index计算出每个卡片的left和right
   initItemStyle(INDEX) {
     return {
        width: this.itemWidth + 'px',
        height: this.itemHeight + 'px',
        left: (INDEX < this.col ? INDEX : (INDEX % this.col)) * this.itemWidth + 'px',
        top: Math.floor(INDEX / this.col) * this.itemHeight + 'px'
     };
   }
 }
```

- 当然我们的卡片数据是从父级传入的，所以`list`肯定会有改变的场景，这时候我们就要重新计算行列数，重新计算容器宽高等，其实也就是重新执行`init`函数；所以我们需要监听`list`；
``` js
  watch: {
    list: {
      handler: function(newVal, oldVal) {
        this.init();
      },
      immediate: true // 定义的时候就执行一次，所以created的时候就不需要执行init了
    }
  },
```

### handleMousedown()
在`handleMousedown()`的时候直接定义`handleMousemove()`和`handleMouseUp()`事件，并且在`handleMouseUp()`中移除；

首先是几个比较重要的变量和方法
- `itemList` ：`list`的拷贝，并加上后续需要用到的属性`dom`（当前卡片的节点信息，通过ref获取）, `isMoveing`（标记当前卡片是否在移动中）, `left`, `top`,
- `curItem` ：当前卡片用的比较多，所以这里单独拿了出来，并且在移动的时候，单前卡片的过渡效果应该去除，不然移动会卡顿，并且`z-index`应该在较高的层级
- `targetItem` ： 即将交换位置的卡片对象，起始为`null`
- `mousePosition` ：鼠标起始位置，移动后的鼠标位置减去起始位置，就是卡片的移动偏移量；

- `handleMousemove()` ：鼠标移动
- `cardDetect()` ：卡片移动检测，是否需要执行位置交换
- `swicthPosition()` ：交换卡片位置
- `handleMouseUp()` ：鼠标抬起

``` js
handleMousedown(e, optionItem) {
  e.preventDefault();
  let that = this;
  if (this.timer) return false; // timer为全局的定时器，表示当前有卡片正在移动，直接返回；

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


  // 鼠标移动
  function handleMousemove(e) {}
  // 卡片交换检测
  function cardDetect() {}
  // 卡片交换
  function swicthPosition() {}
  // 鼠标抬起
  function handleMouseUp() {}
}
```

### handleMousemove(e)
鼠标当前的坐标减去起始的坐标，就是当前卡片的偏移量；

移动过程中就可以执行卡片交换检测，为了提高性能，做了以下节流；200ms执行一次；
``` js
  // 鼠标移动
  function handleMousemove(e) {
    curItem.OffsetLeft = parseInt(e.screenX - mousePosition.startX);
    curItem.OffsetTop = parseInt(e.screenY - mousePosition.startY);
    // 改变当前卡片对应的style
    curItem.dom.style.left = curItem.startLeft + curItem.OffsetLeft + 'px';
    curItem.dom.style.top = curItem.startTop + curItem.OffsetTop + 'px';
    // 卡片交换检测，做一下节流
    if (!DectetTimer) {
      DectetTimer = setTimeout(() => {
        cardDetect();
        clearTimeout(DectetTimer);
        DectetTimer = null;
      }, 200)
    }
  }
```
### cardDetect()
一开始想到的是用碰撞检测去做，循环整个itemList，然后对比当前卡片和每一项的距离；当小于设定的`gap`的时候，就执行`swicthPosition()`；

后面看了`裂泉`的原文章后，发现之前的做法性能差太多了；一直在循环数组；

通过当前的位置和偏移量，可以计算出目标位置`targetItemDragCardIndex`,判断一些临界值之后便执行交换函数；
``` js
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
```

### swicthPosition()
卡片交换分为两种情况；
- 当目标位置比当前移动卡片的原位置大的时候，相隔的卡片和目标卡片都要后移一个位置；
- 当目标位置比当前移动卡片的原位置小的时候，相隔的卡片和目标卡片都要前移一个位置；

::: tip 注意
1. 当我们移动的时候，我们拿的是前一个或者后一个的值，所以我们遍历数组的时候要注意从目标值开始遍历；
2. itemList是list的备份，当我们修改了卡片的dragCard_index之后，需要同步到list中；
3. 卡片交换动画为300ms，这个时间段卡片不应该参与交换检测，所以设置`isMoveing = true`,并设置定时器300ms后清除`isMoveing`
4. 交换卡片过程中，当前卡片只需要改变`itemList`中的属性，不需要改变`list`中，等到最后松开鼠标的时候才同步到`list`中
:::

``` js
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
    // 派发change事件通知父组件
    that.$emit('change', itemList.map(item => item.dragCard_index));
  }
```

### handleMouseUp()
- 当鼠标抬起的时候应该判断是否有目标卡片，如果有的话，就回到目标卡片，没有的话就回到初始位置；
- 当前卡片在鼠标点击的时候去除了过渡效果，当鼠标抬起的时候应该给过渡效果加回去；因为`transition`在`css`中设置了,这里把`style`清除即可
``` js
  function handleMouseUp() {
    //移除所有监听
    document.removeEventListener("mousemove", handleMousemove);
    document.removeEventListener("mouseup", handleMouseUp);

    // 清除检测的定时器并做最后一次碰撞检测
    clearTimeout(DectetTimer);
    DectetTimer = null;
    cardDetect();
    // 把过渡效果加回去
    curItem.dom.style.transition = '';
    // 同步dragCard_index到list中；
    that.list.find(item => item.dragCard_id === optionItem.dragCard_id).dragCard_index = curItem.dragCard_index;
    curItem.dom.style.left = curItem.left + 'px';
    curItem.dom.style.top = curItem.top + 'px';
    // 派发mouseUp事件通知父组件
    that.$emit('mouseUp', that.list.map(item => item.dragCard_index));
    that.timer = setTimeout(() => {
      curItem.dom.style.zIndex = '';
      curItem.dom.childNodes[0].style.boxShadow = 'none';
      clearTimeout(that.timer);
      that.timer = null;
    }, 300);
  }
```

## 写在后面
到这里这个组件就完成啦!

最后贴上来自`裂泉`的原文章链接:  [跟我一起，从0实现并封装拖拽排列组件](https://juejin.im/post/5dae5daae51d4524c24821de) ；这还是一个系列文章，`todo`中后续还会分享如果把组件上传到`npm`;

dranein@163.com

地址：https://github.com/Dranein/vue-study/tree/master/dragCard/src

