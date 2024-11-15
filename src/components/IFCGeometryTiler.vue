<template>
    <div class="ifc-geometry-tiler">
      <h1 class="text-2xl font-bold mb-4">IFC Geometry Tiler</h1>
      <div class="mb-4">
        <input type="file" @change="handleFileSelect" accept=".ifc" class="block w-full text-sm text-gray-500
          file:mr-4 file:py-2 file:px-4
          file:rounded-full file:border-0
          file:text-sm file:font-semibold
          file:bg-violet-50 file:text-violet-700
          hover:file:bg-violet-100
        "/>
      </div>
      <button @click="processIFC" :disabled="!selectedFile || isProcessing" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded disabled:opacity-50 disabled:cursor-not-allowed">
        {{ isProcessing ? 'Processing...' : 'Generate Geometry Tiles' }}
      </button>
      <div v-if="processingStatus" class="mt-4 p-4 bg-gray-100 rounded">
        <h2 class="font-semibold mb-2">Processing Status:</h2>
        <pre class="text-sm">{{ processingStatus }}</pre>
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted } from 'vue';
  import * as OBC from "@thatopen/components";
  import * as THREE from 'three';
  
  const selectedFile = ref<File | null>(null);
  const isProcessing = ref(false);
  const processingStatus = ref('');
  
  const components = new OBC.Components();
  const tiler = components.get(OBC.IfcGeometryTiler);
  
  let files: { name: string; bits: (Uint8Array | string)[] }[] = [];
  let geometriesData: OBC.StreamedGeometries = {};
  let assetsData: OBC.StreamedAsset[] = [];
  let geometryFilesCount = 1;
  
  onMounted(() => {
    setupTiler();
  });
  
  const setupTiler = () => {
    tiler.settings.wasm = {
      path: "https://unpkg.com/web-ifc@0.0.57/",
      absolute: true,
    };
    tiler.settings.minGeometrySize = 20;
    tiler.settings.minAssetsSize = 1000;
  
    tiler.onGeometryStreamed.add(async (geometry) => {
      const { buffer, data } = geometry;
      const bufferFileName = `ifc-processed-geometries-${geometryFilesCount}`;
      for (const expressID in data) {
        const value = data[expressID];
        value.geometryFile = bufferFileName;
        geometriesData[expressID] = value;
      }
      files.push({ name: bufferFileName, bits: [buffer] });
      geometryFilesCount++;
      processingStatus.value += `\nGeometry chunk ${geometryFilesCount - 1} processed`;
    });
  
    tiler.onAssetStreamed.add(async (assets) => {
      assetsData = [...assetsData, ...assets];
      processingStatus.value += `\n${assets.length} assets processed`;
    });
  
    tiler.onIfcLoaded.add(async (groupBuffer) => {
      files.push({
        name: "ifc-processed-global",
        bits: [groupBuffer],
      });
      processingStatus.value += '\nGlobal data processed';
    });
  
    tiler.onProgress.add(async (progress) => {
      if (progress === 1) {
        setTimeout(async () => {
          const processedData = {
            geometries: geometriesData,
            assets: assetsData,
            globalDataFileId: "ifc-processed-global",
          };
          files.push({
            name: "ifc-processed.json",
            bits: [JSON.stringify(processedData)],
          });
          await downloadFilesSequentially(files);
          resetState();
        });
      }
    });
  };
  
  const handleFileSelect = (event: Event) => {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      selectedFile.value = input.files[0];
    }
  };
  
  const processIFC = async () => {
    if (!selectedFile.value) return;
  
    isProcessing.value = true;
    processingStatus.value = 'Starting IFC processing...';
  
    try {
      const ifcBuffer = await selectedFile.value.arrayBuffer();
      const ifcArrayBuffer = new Uint8Array(ifcBuffer);
      await tiler.streamFromBuffer(ifcArrayBuffer);
    } catch (error: any) {
      processingStatus.value += `\nError: ${error.message}`;
    } finally {
      isProcessing.value = false;
    }
  };
  
  const downloadFile = (name: string, ...bits: (Uint8Array | string)[]) => {
    const blob = new Blob(bits, { type: 'application/octet-stream' });
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = name;
    anchor.click();
    URL.revokeObjectURL(url);
  };
  
  const downloadFilesSequentially = async (fileList: { name: string; bits: (Uint8Array | string)[] }[]) => {
    for (const { name, bits } of fileList) {
      downloadFile(name, ...bits);
      await new Promise((resolve) => {
        setTimeout(resolve, 100);
      });
    }
    processingStatus.value += '\nAll files downloaded';
  };
  
  const resetState = () => {
    assetsData = [];
    geometriesData = {};
    files = [];
    geometryFilesCount = 1;
    processingStatus.value += '\nProcessing complete';
  };
  </script>