<template>
  <VideoCall/>
</template>

<script>
import VideoCall from './components/VideoCall.vue'

export default {
  name: 'App',
  components: {
    VideoCall
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
}
</style>
