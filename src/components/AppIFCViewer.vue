<template>
  <div class="ifc-viewer-container">
    <div id="ifc-container" class="h-full" aria-label="IFC Model Viewer">
      <!-- The IFC model will be rendered here -->
    </div>
    <div v-if="loading" class="loading-overlay" aria-live="polite">
      <div class="loading-spinner" role="progressbar" aria-valuetext="Loading"></div>
      <p>Loading IFC Model...</p>
    </div>
    <div v-if="selectedElement" class="element-properties">
      <h2>Element Properties</h2>
      <pre>{{ JSON.stringify(selectedElement, null, 2) }}</pre>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, defineProps } from 'vue';
import * as OBC from '@thatopen/components';
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';
import * as FRAGS from "@thatopen/fragments";
import * as OBF from "@thatopen/components-front";

const props = defineProps({
  URL: { type: String, required: true },
  META: { type: String, required: true },
  fileName: { type: String, default: '' }
});

const world = ref<OBC.SimpleWorld<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer> | null>(null);
let resizeObserver: ResizeObserver | null = null;
const loading = ref(false);
const selectedElement = ref(null);
let model: FRAGS.FragmentsGroup;
let ifcData: any;
let highlighter: OBF.Highlighter;
let relationsTree: any;

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

    const [tree, setRelationsTree] = CUI.tables.relationsTree({
      components,
      models: [],
    });

    relationsTree = tree;
    relationsTree.preserveStructureOnFilter = true;

    highlighter = components.get(OBF.Highlighter);
    highlighter.setup({ world: newWorld });
    highlighter.zoomToSelection = true;

    highlighter.events.select.onHighlight.add((selection: any) => {
      if (Object.keys(selection).length > 0) {
        const firstKey = Object.keys(selection)[0];
        const expressID = selection[firstKey].values().next().value;
        console.log('expressID : ', expressID)
        const elementData = ifcData.find((item: any) => item.expressID === expressID);
        selectedElement.value = elementData;
      } else {
        selectedElement.value = null;
      }
    });

    const panel = BUI.Component.create(() => {
      const onSearch = (e: Event) => {
        const input = e.target as BUI.TextInput;
        relationsTree.queryString = input.value;
      };

      return BUI.html`
      <bim-panel label="Relations Tree">
        <bim-panel-section label="Model Tree">
          <bim-text-input @input=${onSearch} placeholder="Search..." debounce="200"></bim-text-input>
          ${relationsTree}
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
          / 30rem 1fr
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

    const indexer = world.value.components.get(OBC.IfcRelationsIndexer);
    await indexer.process(model);

    // Load IFC data
    const ifcResponse = await fetch(props.META);
    const rawIfcData = await ifcResponse.json();

    ifcData = Array.isArray(rawIfcData) ? rawIfcData : Object.values(rawIfcData);

    // Update the relationsTree with the loaded model
    if (relationsTree) {
      relationsTree.models = [model];
      relationsTree.update();
    }

    // Set up event listener for relationsTree selection
    relationsTree.onSelected = (expressID: number) => {
      // highlighter.highlightByID("select", model.id, [expressID]);
    };

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
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

.element-properties {
  position: absolute;
  top: 20px;
  right: 20px;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 20px;
  border-radius: 5px;
  max-width: 300px;
  max-height: 80%;
  overflow-y: auto;
  text-align: left;
}
</style>