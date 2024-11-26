<template>
  <div class="ifc-viewer-container">
    <div id="ifc-container" class="h-full" aria-label="IFC Model Viewer">
      <!-- The IFC model will be rendered here -->
    </div>
    <div v-if="loading" class="loading-overlay" aria-live="polite">
      <div class="loading-spinner" role="progressbar" aria-valuetext="Loading"></div>
      <p>Loading IFC Model...</p>
    </div>

    <!-- <button @click="loadPredefined" :disabled="loading" class="load-button">
      {{ loading ? 'Loading...' : 'Load Predefinded Model' }}
    </button> -->
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, defineProps } from 'vue';
import * as OBC from '@thatopen/components';
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';
import * as FRAGS from "@thatopen/fragments";

const props = defineProps({
  URL: { type: String, required: true },
  fileName: { type: String, default: '' }
});

const world = ref<OBC.SimpleWorld<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer> | null>(null);
let resizeObserver: ResizeObserver | null = null;
const loading = ref(false);
let updateClassificationsTree: (arg: { classifications: any[] }) => void;
let model: FRAGS.FragmentsGroup;
let classifier: OBC.Classifier;

const initializeWorld = () => {
  BUI.Manager.init();
  const viewport = document.createElement('bim-viewport');
  viewport.setAttribute('aria-label', 'IFC Model Viewport');
  const container = document.getElementById('ifc-container');
  const components = new OBC.Components();

  if (container) {
    const worlds = components.get(OBC.Worlds);
    const newWorld = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();
    newWorld.scene = new OBC.SimpleScene(components);
    newWorld.renderer = new OBC.SimpleRenderer(components, viewport);
    newWorld.camera = new OBC.SimpleCamera(components);

    components.init();
    newWorld.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);
    newWorld.scene.setup();

    const cullers = components.get(OBC.Cullers);
    const culler = cullers.create(newWorld);
    culler.config.threshold = 200;
    culler.needsUpdate = true;

    newWorld.camera.controls.addEventListener('controlend', () => {
      culler.needsUpdate = true;
    });

    const grids = components.get(OBC.Grids);
    grids.create(newWorld);

    const fragmentIfcLoader = components.get(OBC.IfcLoader);
    fragmentIfcLoader.settings.wasm = {
      path: 'https://unpkg.com/web-ifc@0.0.61/',
      absolute: true
    };
    fragmentIfcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
    (fragmentIfcLoader.settings.webIfc as any).OPTIMIZE_PROFILES = true;

    const [classificationsTree, updateClassificationsTreeFunc] = CUI.tables.classificationTree({
      components,
      classifications: []
    });

    updateClassificationsTree = updateClassificationsTreeFunc;

    const panel = BUI.Component.create(() => {
      return BUI.html`
        <bim-panel label="Classifications Tree">
          <bim-panel-section label="Classifications">
            ${classificationsTree}
          </bim-panel-section>
        </bim-panel>
      `;
    });

    const app = document.createElement('bim-grid');
    app.setAttribute('role', 'application');
    app.setAttribute('aria-label', 'IFC Model Viewer Application');
    app.layouts = {
      main: {
        template: `
          "panel viewport"
          / 23rem 1fr
        `,
        elements: { panel, viewport }
      }
    };
    app.layout = 'main';
    container.append(app);

    world.value = newWorld;

    resizeObserver = new ResizeObserver(
      debounce(() => {
        if (newWorld.renderer && newWorld.camera) {
          newWorld.renderer.resize();
          newWorld.camera.updateAspect();
        }
      }, 100)
    );
    resizeObserver.observe(viewport);
  }
};

const loadModel = async () => {
  if (!world.value || !props.URL) return;

  loading.value = true;
  try {
    const fragmentIfcLoader = world.value.components.get(OBC.IfcLoader);
    const response = await fetch(props.URL);
    const data = await response.arrayBuffer();
    const buffer = new Uint8Array(data);
    model = await fragmentIfcLoader.load(buffer);
    world.value.scene.three.add(model);

    classifier = world.value.components.get(OBC.Classifier);

    classifier.byModel(model.uuid, model);
    classifier.byEntity(model);

    const classifications = [
      { system: 'entities', label: 'Entities' },
      { system: 'predefinedTypes', label: 'Predefined Types' },
    ];

    updateClassificationsTree({ classifications });
  } catch (error) {
    console.error('Error loading IFC model:', error);
  } finally {
    loading.value = false;
  }
};

const cleanupMemory = () => {
  if (world.value) {
    const fragmentManager = world.value.components.get(OBC.FragmentsManager);
    fragmentManager.dispose();
  }
  if (model) {
    model.dispose();
  }
  // Force garbage collection if available
  if (typeof global !== 'undefined' && global.gc) {
    global.gc();
  }
};

const debounce = (func: any, wait: number) => {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  return (...args: any[]) => {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
};

onMounted(() => {
  initializeWorld();
  // Automatically load the model after initialization
  loadModel();
});

onUnmounted(() => {
  cleanupMemory();
  world.value?.dispose();
  if (resizeObserver) {
    resizeObserver.disconnect();
  }
});

watch(() => props.URL, loadModel);
</script>

<style scoped>
.ifc-viewer-container {
  position: relative;
  height: 100%;
  width: 100%;
}

.h-full {
  height: 100%;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
  font-size: 1.2rem;
}

.loading-spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.load-button {
  position: absolute;
  top: 20px;
  right: 20px;
  padding: 10px 20px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1rem;
}

.load-button:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}

.classification-progress {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px;
  border-radius: 5px;
  font-size: 14px;
}
</style>