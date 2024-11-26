<template>
  <div id="app">
    <AppIFCViewer :URL="'http://localhost:8080/small.ifc'" />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-facing-decorator';
import AppIFCViewer from './components/AppIFCViewer.vue';

@Component({
  components: {
    AppIFCViewer
  }
})
export default class App extends Vue {
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  height: 100vh;
}

#viewer {
  height: 100vh;
}

body {
  margin: 0;
}
</style>