<template>
  <div id="app">
    <!-- <IfcRelationsDownloader /> -->
     <!-- <IfcJsonExporter /> -->
    <AppIFCViewer :URL="'http://localhost:8080/small.ifc'" :META="'http://localhost:8080/properties.json'" highlighId=210682 />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-facing-decorator';
import AppIFCViewer from './components/AppIFCViewer.vue';
import IfcRelationsDownloader from './components/IfcRelationsDownloader.vue'
import IfcJsonExporter from './components/IfcJsonExporter.vue'

@Component({
  components: {
    AppIFCViewer,
    IfcRelationsDownloader,
    IfcJsonExporter
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