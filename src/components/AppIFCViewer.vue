<template>
  <div id="ifc-container" class="h-full" aria-label="IFC Model Viewer">
    <!-- The IFC model will be rendered here -->
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, defineProps } from 'vue';
import * as OBC from '@thatopen/components';
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';

const props = defineProps({
  URL: { type: String, default: '' },
  fileName: { type: String, default: '' }
});

const world = ref<OBC.SimpleWorld<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer> | null>(null);
let resizeObserver: ResizeObserver | null = null;

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

    const classifier = components.get(OBC.Classifier);
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

    const [classificationsTree, updateClassificationsTree] = CUI.tables.classificationTree({
      components,
      classifications: []
    });

    // const fragmentsManager = components.get(OBC.FragmentsManager);
    // fragmentsManager.onFragmentsLoaded.add((model: any) => {
    //   classifier.byEntity(model);
    //   classifier.byPredefinedType(model).then(() => {
    //     console.timeEnd('Loading model properties Time');
    //     const classifications = [
    //       { system: 'entities', label: 'Entities' },
    //       { system: 'predefinedTypes', label: 'Predefined Types' }
    //     ];
    //     updateClassificationsTree({ classifications });
    //   });
    // });

    if (props.URL) {
      fetch(props.URL)
        .then(response => response.arrayBuffer())
        .then(data => {
          const buffer = new Uint8Array(data);
          return fragmentIfcLoader.load(buffer);
        })
        .then(model => {
          newWorld.scene.three.add(model);
        })
        .catch(error => {
          console.error('Error loading IFC model:', error);
        });

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
    }

    world.value = newWorld;

    // Set up ResizeObserver
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

const debounce = (func: any, wait: number) => {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  return (...args: any[]) => {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
};

onMounted(initializeWorld);

onUnmounted(() => {
  world.value?.dispose();
  if (resizeObserver) {
    resizeObserver.disconnect();
  }
});

watch(() => props.URL, initializeWorld);
</script>

<style scoped>
.h-full {
  height: 100%;
}
</style>