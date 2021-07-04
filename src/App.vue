<template>
  <websocket msg="Welcome to Your Vue.js App"/>
</template>

<script>
import websocket from './components/Websocket.vue';

export default {
  name: 'App',
  components: {
    websocket,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  padding:10px;
  display:flex;
  justify-content:center;
}
</style>
