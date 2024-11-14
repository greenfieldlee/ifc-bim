<template>
  <div id="app">
    <h1>IFC Viewer</h1>
    <AppIFCViewer :URL="ifcFileUrl" :fileName="ifcFileName" />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import AppIFCViewer from './components/AppIFCViewer.vue'

export default defineComponent({
  name: 'App',
  components: {
    AppIFCViewer
  },
  data() {
    return {
      // ifcFileUrl: './DC_Riverside-STRUC-LOD_300.ifc',
      ifcFileUrl: './test-1.ifc',
      ifcFileName: 'example.ifc'
    }
  }
})
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