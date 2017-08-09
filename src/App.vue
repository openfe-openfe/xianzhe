<template>
  <div id="app">
  <keep-alive>
      <router-view></router-view>
  </keep-alive>
  <m-footer></m-footer>
  </div>
</template>

<script>
  import MFooter from 'components/m-footer/m-footer'
  export default {
    components: {
      MFooter
    }
  }
</script>

