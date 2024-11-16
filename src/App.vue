<template>
  <div id="app">
    <h1>IFC Viewer</h1>
    <IFCTileConverter />
    <AppIFCStreamingViewer
      :geometryURL="geometryURL"
      :propertiesURL="propertiesURL"
      id="viewer"
    />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-facing-decorator';
import AppIFCStreamingViewer from './components/AppIFCStreamingViewer.vue';
import IFCTileConverter from './components/IFCTileConverter.vue';

@Component({
  components: {
    AppIFCStreamingViewer,
    IFCTileConverter
  }
})
export default class App extends Vue {
  geometryURL = 'http://localhost:8080/ifc-processed.json';
  propertiesURL = 'http://localhost:8080/ifc-processed-properties.json';
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
  height: 100vh;
}

#viewer {
  height: 80vh;
}
</style>