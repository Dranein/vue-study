<template>
  <div id="app">
    <DragCard
      :list="list"
      :col="4"
      :itemWidth="150"
      :itemHeight="150"
      @change="handleChange"
      @mouseUp="handleMouseUp">
      <!-- <template v-slot:head="{ item }">
        <div class="dragHead">{{item.head}}</div>
      </template>
      <template v-slot:content="{ item }">
        <div class="dragContent">{{item.content}}</div>
      </template> -->
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
          {head: '标题2', content: "演示卡片2"},
          {head: '标题3', content: "演示卡片3"},
          {head: '标题4', content: "演示卡片4"},
          {head: '标题5', content: "演示卡片5"},
          {head: '标题6', content: "演示卡片6"},
          {head: '标题7', content: "演示卡片7"},
          {head: '标题8', content: "演示卡片8"},
          {head: '标题9', content: "演示卡片9"}
        ]
      }
    },
    mounted () {

    },
    methods: {
      handleChange(data) {
      },
      handleMouseUp(data) {
      }
    }
  }
</script>

<style lang="scss" scoped="">
  #app {
    background: #ddd;
  }
  .dragHead {
    height: 40px;
    background: #8a977b;
    color: #fff;
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    padding-left: 20px;
  }
  .dragContent {
    background: #b6c29a;
    color: #eee;
    font-size: 16px;
    height: 100%;
    width: 100%;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
  }
</style>
