<template>
    <div class="ifc-properties-tiler">
      <h1 class="text-2xl font-bold mb-4">IFC Properties Tiler</h1>
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
        {{ isProcessing ? 'Processing...' : 'Generate Property Tiles' }}
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
  
  interface StreamedProperties {
    types: {
      [typeID: number]: number[];
    };
    ids: {
      [id: number]: number;
    };
    indexesFile: string;
  }
  
  const selectedFile = ref<File | null>(null);
  const isProcessing = ref(false);
  const processingStatus = ref('');
  
  const components = new OBC.Components();
  const propsStreamer = components.get(OBC.IfcPropertiesTiler);
  
  let counter = 0;
  const files: { name: string; bits: Blob }[] = [];
  const jsonFile: StreamedProperties = {
    types: {},
    ids: {},
    indexesFile: "ifc-processed-properties-indexes",
  };
  
  onMounted(() => {
    setupPropsStreamer();
  });
  
  const setupPropsStreamer = () => {
    propsStreamer.settings.wasm = {
      path: "https://unpkg.com/web-ifc@0.0.57/",
      absolute: true,
    };
  
    propsStreamer.onPropertiesStreamed.add(async (props) => {
      if (!jsonFile.types[props.type]) {
        jsonFile.types[props.type] = [];
      }
      jsonFile.types[props.type].push(counter);
  
      for (const id in props.data) {
        jsonFile.ids[id] = counter;
      }
  
      const name = `ifc-processed-properties-${counter}`;
      const bits = new Blob([JSON.stringify(props.data)]);
      files.push({ bits, name });
  
      counter++;
      processingStatus.value += `\nProcessed property chunk ${counter}`;
    });
  
    propsStreamer.onProgress.add(async (progress) => {
      processingStatus.value = `Processing: ${Math.round(progress * 100)}%`;
    });
  
    propsStreamer.onIndicesStreamed.add(async (props) => {
      files.push({
        name: `ifc-processed-properties.json`,
        bits: new Blob([JSON.stringify(jsonFile)]),
      });
  
      const relations = components.get(OBC.IfcRelationsIndexer);
      const serializedRels = relations.serializeRelations(props);
  
      files.push({
        name: "ifc-processed-properties-indexes",
        bits: new Blob([serializedRels]),
      });
  
      await downloadFilesSequentially(files);
      processingStatus.value += '\nProcessing complete. Files downloaded.';
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
      await propsStreamer.streamFromBuffer(ifcArrayBuffer);
    } catch (error: any) {
      processingStatus.value += `\nError: ${error.message}`;
    } finally {
      isProcessing.value = false;
    }
  };
  
  const downloadFile = (name: string, bits: Blob) => {
    const url = URL.createObjectURL(bits);
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = name;
    anchor.click();
    URL.revokeObjectURL(url);
  };
  
  const downloadFilesSequentially = async (fileList: { name: string; bits: Blob }[]) => {
    for (const { name, bits } of fileList) {
      downloadFile(name, bits);
      await new Promise((resolve) => {
        setTimeout(resolve, 100);
      });
    }
  };
  </script>