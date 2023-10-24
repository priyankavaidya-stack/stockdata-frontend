<template>
  <UserInput msg="Stock Market Data Visualization"/>
</template>

<script>
import UserInput from './components/UserInput.vue';

export default {
  name: 'App',
  components: {
    UserInput
}
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #fff;
  background-color: #040a11;
  height: 120vh;
  width: 100%;
}
</style>
