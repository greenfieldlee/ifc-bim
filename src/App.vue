<template>
  <div id="app">
    <h1>IFC Viewer</h1>
    <!-- <IFCGeometryTiler />
    <IFCPropertiesTiler /> -->
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
import IFCGeometryTiler from './components/IFCGeometryTiler.vue';
import IFCPropertiesTiler from './components/IFCPropertiesTiler.vue';

@Component({
  components: {
    AppIFCStreamingViewer,
    IFCGeometryTiler,
    IFCPropertiesTiler
  }
})
export default class App extends Vue {
  // geometryURL = 'http://thatopen.github.io/engine_components/resources/streaming/small.ifc-processed.json';
  // propertiesURL = 'https://thatopen.github.io/engine_components/resources/streaming/small.ifc-processed-properties.json';
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