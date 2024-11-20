<template>
  <div id="app">
    <!-- <IFCTileConverter />
    <AppIFCStreamingViewer
      :baseURL="baseURL"
      :geometryURL="geometryURL"
      :propertiesURL="propertiesURL"
      id="viewer"
    />  -->
    <AppIFCViewer :URL="'http://localhost:8080/big.ifc'" />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-facing-decorator';
import AppIFCStreamingViewer from './components/AppIFCStreamingViewer.vue';
import IFCTileConverter from './components/IFCTileConverter.vue';
import AppIFCViewer from './components/AppIFCViewer.vue';

@Component({
  components: {
    AppIFCStreamingViewer,
    IFCTileConverter,
    AppIFCViewer
  }
})
export default class App extends Vue {
  baseURL = process.env.VUE_APP_API_URL;
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
  height: 100vh;
}

#viewer {
  height: 100vh;
}

body {
  margin: 0;
}
</style>