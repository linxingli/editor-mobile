<template>
  <div id="app">
    <edit-dom v-model="content"
              ref="editdom"
              placeholder="请输入" />

    <van-button class="btn"
                @click="open">打开富文本面板</van-button>
  </div>
</template>

<script>
  import editDom from './editDom/index.vue'
  export default {
    name: 'App',
    components: {
      editDom
    },
    data () {
      return {
        content: ''
      }
    },
    methods: {
      open () {
        this.$refs['editdom'].showToolbar() // 弹出富文本面板
      }
    }
  }
</script>

<style>
  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
    padding: 0 30px;
  }
  .btn {
    margin-top: 50px !important;
  }
</style>
