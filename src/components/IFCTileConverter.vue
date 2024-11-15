<template>
    <div class="ifc-tile-converter">
      <h2>IFC Tile Converter</h2>
      <input type="file" @change="handleFileSelect" accept=".ifc" />
      <button @click="convertToTiles" :disabled="!selectedFile">Convert to Tiles</button>
      <div v-if="conversionProgress > 0 && conversionProgress < 1">
        Conversion Progress: {{ (conversionProgress * 100).toFixed(2) }}%
      </div>
      <div v-if="conversionComplete">
        Conversion complete. Files are ready for download.
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted, defineExpose } from 'vue';
  import * as OBC from '@thatopen/components';
  
  interface GeometryData {
    geometries: { [key: string]: any };
    assets: OBC.StreamedAsset[];
  }
  
  interface PropertyData {
    types: { [typeID: number]: number[] };
    ids: { [id: number]: number };
    indexesFile: string;
    properties: { [id: string]: any };
  }
  
  const selectedFile = ref<File | null>(null);
  const conversionProgress = ref(0);
  const conversionComplete = ref(false);
  const components = ref<OBC.Components | null>(null);
  const geometryTiler = ref<OBC.IfcGeometryTiler | null>(null);
  const propertyTiler = ref<OBC.IfcPropertiesTiler | null>(null);
  
  onMounted(() => {
    components.value = new OBC.Components();
    geometryTiler.value = components.value.get(OBC.IfcGeometryTiler);
    propertyTiler.value = components.value.get(OBC.IfcPropertiesTiler);
    initializeTilers();
  });
  
  const initializeTilers = () => {
    const wasmSettings = {
      path: "https://unpkg.com/web-ifc@0.0.57/",
      absolute: true,
    };
  
    if (geometryTiler.value) {
      geometryTiler.value.settings.wasm = wasmSettings;
      geometryTiler.value.settings.minGeometrySize = 20;
      geometryTiler.value.settings.minAssetsSize = 1000;
    }
  
    if (propertyTiler.value) {
      propertyTiler.value.settings.wasm = wasmSettings;
    }
  };
  
  const handleFileSelect = (event: Event) => {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      selectedFile.value = input.files[0];
    }
  };
  
  const downloadFile = (name: string, content: string) => {
    const blob = new Blob([content], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = name;
    a.click();
    URL.revokeObjectURL(url);
  };
  
  const convertToTiles = async () => {
    if (!selectedFile.value) return;
  
    const ifcBuffer = await selectedFile.value.arrayBuffer();
    const ifcArrayBuffer = new Uint8Array(ifcBuffer);
  
    await convertGeometry(ifcArrayBuffer);
    await convertProperties(ifcArrayBuffer);
  
    conversionComplete.value = true;
    conversionProgress.value = 0;
  };
  
  const convertGeometry = async (ifcArrayBuffer: Uint8Array): Promise<void> => {
    let currentGeometryFile: GeometryData = { geometries: {}, assets: [] };
    let fileCounter = 0;
  
    if (geometryTiler.value) {
      geometryTiler.value.onGeometryStreamed.add(async (data: { buffer: Uint8Array; data: OBC.StreamedGeometries }) => {
        const { data: geometryItems } = data;
        for (const expressID in geometryItems) {
          if (Object.prototype.hasOwnProperty.call(geometryItems, expressID)) {
            currentGeometryFile.geometries[expressID] = geometryItems[expressID];
          }
        }
  
        // If the current file size exceeds a threshold, save it and start a new one
        if (JSON.stringify(currentGeometryFile).length > 1000000) { // 1MB threshold
          downloadFile(`${selectedFile.value!.name}-geometry-${fileCounter}.json`, JSON.stringify(currentGeometryFile));
          currentGeometryFile = { geometries: {}, assets: [] };
          fileCounter++;
        }
      });
  
      geometryTiler.value.onAssetStreamed.add(async (assets: OBC.StreamedAsset[]) => {
        currentGeometryFile.assets.push(...assets);
      });
  
      geometryTiler.value.onIfcLoaded.add(async (data: any) => {
        downloadFile(`${selectedFile.value!.name}-global-data.json`, JSON.stringify(data));
      });
  
      geometryTiler.value.onProgress.add(async (progress: number) => {
        conversionProgress.value = progress / 2;
      });
  
      await geometryTiler.value.streamFromBuffer(ifcArrayBuffer);
  
      // Save any remaining geometry data
      if (Object.keys(currentGeometryFile.geometries).length > 0 || currentGeometryFile.assets.length > 0) {
        downloadFile(`${selectedFile.value!.name}-geometry-${fileCounter}.json`, JSON.stringify(currentGeometryFile));
      }
    }
  };
  
  const convertProperties = async (ifcArrayBuffer: Uint8Array): Promise<void> => {
    const propertyData: PropertyData = {
      types: {},
      ids: {},
      indexesFile: `${selectedFile.value!.name}-processed-properties-indexes.json`,
      properties: {},
    };
  
    if (propertyTiler.value) {
      propertyTiler.value.onPropertiesStreamed.add(async (props: any) => {
        if (!propertyData.types[props.type]) {
          propertyData.types[props.type] = [];
        }
        propertyData.types[props.type].push(Object.keys(propertyData.properties).length);
  
        for (const id in props.data) {
          if (Object.prototype.hasOwnProperty.call(props.data, id)) {
            propertyData.ids[Number(id)] = Object.keys(propertyData.properties).length;
            propertyData.properties[id] = props.data[id];
          }
        }
      });
  
      propertyTiler.value.onIndicesStreamed.add(async (props: any) => {
        const relations = components.value!.get(OBC.IfcRelationsIndexer);
        const serializedRels = relations.serializeRelations(props);
        propertyData.indexesFile = JSON.stringify(serializedRels);
      });
  
      propertyTiler.value.onProgress.add(async (progress: number) => {
        conversionProgress.value = 0.5 + progress / 2;
      });
  
      await propertyTiler.value.streamFromBuffer(ifcArrayBuffer);
  
      downloadFile(`${selectedFile.value!.name}-properties.json`, JSON.stringify(propertyData));
    }
  };
  
  defineExpose({
    convertToTiles,
    conversionProgress,
    conversionComplete
  });
  </script>
  
  <style scoped>
  .ifc-tile-converter {
    padding: 20px;
    border: 1px solid #ccc;
    border-radius: 5px;
    margin-bottom: 20px;
  }
  
  button {
    margin-top: 10px;
    padding: 5px 10px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 3px;
    cursor: pointer;
  }
  
  button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
  </style>