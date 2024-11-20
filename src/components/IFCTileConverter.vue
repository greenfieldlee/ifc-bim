<template>
  <div class="combined-ifc-tiler">
    <h1 class="text-2xl font-bold mb-4">Combined IFC Tiler</h1>
    <div class="mb-4">
      <input type="file" @change="handleFileSelect" accept=".ifc" class="block w-full text-sm text-gray-500
          file:mr-4 file:py-2 file:px-4
          file:rounded-full file:border-0
          file:text-sm file:font-semibold
          file:bg-violet-50 file:text-violet-700
          hover:file:bg-violet-100
        " aria-label="Select IFC file" />
      <button @click="processIFC" :disabled="!selectedFile || isProcessing"
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded disabled:opacity-50 disabled:cursor-not-allowed">
        {{ isProcessing ? 'Processing...' : 'Generate Tiles' }}
      </button>
    </div>
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
const geometryTiler = components.get(OBC.IfcGeometryTiler);
const propertyTiler = components.get(OBC.IfcPropertiesTiler);

let geometryFiles: { name: string; bits: (Uint8Array | string)[] }[] = [];
let propertyFiles: { name: string; bits: (Uint8Array | string)[] }[] = [];
let geometriesData: OBC.StreamedGeometries = {};
let assetsData: OBC.StreamedAsset[] = [];
let geometryFilesCount = 1;
let propertyCounter = 0;

const propertyJsonFile: StreamedProperties = {
  types: {},
  ids: {},
  indexesFile: "ifc-processed-properties-indexes",
};

onMounted(() => {
  setupTilers();
});

const setupTilers = () => {
  const wasmSettings = {
    path: "https://unpkg.com/web-ifc@0.0.57/",
    absolute: true,
  };

  // Setup Geometry Tiler
  geometryTiler.settings.wasm = wasmSettings;
  geometryTiler.settings.minGeometrySize = 20;
  geometryTiler.settings.minAssetsSize = 1000;

  geometryTiler.onGeometryStreamed.add(async (geometry) => {
    const { buffer, data } = geometry;
    const bufferFileName = `ifc-processed-geometries-${geometryFilesCount}`;
    for (const expressID in data) {
      const value = data[expressID];
      value.geometryFile = bufferFileName;
      geometriesData[expressID] = value;
    }
    geometryFiles.push({ name: bufferFileName, bits: [buffer] });
    geometryFilesCount++;
    processingStatus.value += `\nGeometry chunk ${geometryFilesCount - 1} processed`;
  });

  geometryTiler.onAssetStreamed.add(async (assets) => {
    assetsData = [...assetsData, ...assets];
    processingStatus.value += `\n${assets.length} geometry assets processed`;
  });

  geometryTiler.onIfcLoaded.add(async (groupBuffer) => {
    geometryFiles.push({
      name: "ifc-processed-global",
      bits: [groupBuffer],
    });
    processingStatus.value += '\nGlobal geometry data processed';
  });

  geometryTiler.onProgress.add(async (progress) => {
    processingStatus.value = `Geometry Processing: ${Math.round(progress * 100)}%`;
  });

  // Setup Property Tiler
  propertyTiler.settings.wasm = wasmSettings;

  propertyTiler.onPropertiesStreamed.add(async (props) => {
    if (!propertyJsonFile.types[props.type]) {
      propertyJsonFile.types[props.type] = [];
    }
    propertyJsonFile.types[props.type].push(propertyCounter);

    for (const id in props.data) {
      propertyJsonFile.ids[id] = propertyCounter;
    }

    const name = `ifc-processed-properties-${propertyCounter}`;
    const bits = [JSON.stringify(props.data)];
    propertyFiles.push({ bits, name });

    propertyCounter++;
    processingStatus.value += `\nProperty chunk ${propertyCounter} processed`;
  });

  propertyTiler.onProgress.add(async (progress) => {
    processingStatus.value = `Property Processing: ${Math.round(progress * 100)}%`;
  });

  propertyTiler.onIndicesStreamed.add(async (props) => {
    propertyFiles.push({
      name: `ifc-processed-properties.json`,
      bits: [JSON.stringify(propertyJsonFile)],
    });

    const relations = components.get(OBC.IfcRelationsIndexer);
    const serializedRels = relations.serializeRelations(props);

    propertyFiles.push({
      name: "ifc-processed-properties-indexes",
      bits: [serializedRels],
    });

    processingStatus.value += '\nProperty processing complete';
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

    processingStatus.value += '\nProcessing geometry...';
    await geometryTiler.streamFromBuffer(ifcArrayBuffer);

    processingStatus.value += '\nProcessing properties...';
    await propertyTiler.streamFromBuffer(ifcArrayBuffer);

    processingStatus.value += '\nGenerating final geometry data...';
    const processedGeometryData = {
      geometries: geometriesData,
      assets: assetsData,
      globalDataFileId: "ifc-processed-global",
    };
    geometryFiles.push({
      name: "ifc-processed.json",
      bits: [JSON.stringify(processedGeometryData)],
    });

    processingStatus.value += '\nDownloading files...';
    await downloadFilesSequentially([...geometryFiles, ...propertyFiles]);

    processingStatus.value += '\nProcessing complete. All files downloaded.';
  } catch (error: any) {
    processingStatus.value += `\nError: ${error.message}`;
  } finally {
    isProcessing.value = false;
    resetState();
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
};


const resetState = () => {
  geometryFiles = [];
  propertyFiles = [];
  geometriesData = {};
  assetsData = [];
  geometryFilesCount = 1;
  propertyCounter = 0;
  propertyJsonFile.types = {};
  propertyJsonFile.ids = {};
};
</script>