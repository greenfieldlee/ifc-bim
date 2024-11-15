<template>
  <div>
    <div id="ifc-container" class="h-full"></div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, toNative, Vue } from 'vue-facing-decorator';
import * as OBC from '@thatopen/components';
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';
import { Group, Object3D, AmbientLight, DirectionalLight } from 'three';

function debounceRAF(func: (...args: any[]) => void): (...args: any[]) => void {
  let rafId: number | null = null;
  let lastArgs: any[] | null = null;

  return (...args: any[]) => {
    lastArgs = args;

    if (rafId) {
      return;
    }

    rafId = requestAnimationFrame(() => {
      if (lastArgs) {
        func(...lastArgs);
      }
      rafId = null;
      lastArgs = null;
    });
  };
}

@Component
class AppIFCViewer extends Vue {
  @Prop({ default: '', type: String }) URL!: string;
  @Prop({ default: '', type: String }) fileName!: string;
  world?: OBC.SimpleWorld<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>;
  resizeHandler?: (...args: any[]) => void;

  async mounted() {
    BUI.Manager.init();
    const viewport = document.createElement('bim-viewport');
    const container = document.getElementById('ifc-container');
    const components = new OBC.Components();

    if (container) {
      const worlds = components.get(OBC.Worlds);
      const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();
      world.scene = new OBC.SimpleScene(components);
      world.renderer = new OBC.SimpleRenderer(components, viewport);

      // Initialize the camera
      world.camera = new OBC.SimpleCamera(components);
      console.log('Camera initialized:', world.camera);

      this.resizeHandler = debounceRAF(() => {
        if (world.renderer && world.camera) {
          world.renderer.resize();
          world.camera.updateAspect();
        }
      });

      viewport.addEventListener('resize', this.resizeHandler);

      components.init();
      world.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);
      world.scene.setup();

      // Add lighting
      const ambientLight = new AmbientLight(0xffffff, 0.5);
      world.scene.three.add(ambientLight);

      const directionalLight = new DirectionalLight(0xffffff, 1);
      directionalLight.position.set(10, 10, 10);
      world.scene.three.add(directionalLight);

      const classifier = components.get(OBC.Classifier);
      const cullers = components.get(OBC.Cullers);
      const culler = cullers.create(world);
      culler.config.threshold = 900;
      culler.needsUpdate = true;

      world.camera.controls.addEventListener('controlend', () => {
        culler.needsUpdate = true;
      });

      const grids = components.get(OBC.Grids);
      grids.create(world);

      const [classificationsTree, updateClassificationsTree] = CUI.tables.classificationTree({
        components,
        classifications: []
      });

      const fragmentsManager = components.get(OBC.FragmentsManager);
      fragmentsManager.onFragmentsLoaded.add(async (model: any) => {
        console.log('Fragments loaded:', model);
        classifier.byEntity(model);
        await classifier.byPredefinedType(model);
        const classifications = [
          { system: 'entities', label: 'Entities' },
          { system: 'predefinedTypes', label: 'Predefined Types' }
        ];
        updateClassificationsTree({ classifications });
      });

      const fragmentIfcLoader = components.get(OBC.IfcLoader);
      fragmentIfcLoader.settings.wasm = {
        path: 'https://unpkg.com/web-ifc@0.0.61/',
        absolute: true
      };
      fragmentIfcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
      (fragmentIfcLoader.settings.webIfc as any).OPTIMIZE_PROFILES = true;

      if (this.URL) {
        const file = await fetch(this.URL);
        if (file) {
          try {

            const data = await file.arrayBuffer();
            const buffer = new Uint8Array(data);
            const model = await fragmentIfcLoader.load(buffer);
            world.scene.three.add(model);
            // console.log('model -> ', model)
            
            
            // Create and set up the worker
            // const worker = new Worker(new URL('./ifc-worker.ts', import.meta.url), { type: 'module' });
            // worker.postMessage({ data: buffer, wasmPath: 'https://unpkg.com/web-ifc@0.0.61/'});

            // worker.onmessage = (event) => {
            //   console.log('onmessage', event.data);
            //   // const { model, error } = event.data;
            //   const { error } = event.data;
            //   if (error) {
            //     console.error('Error loading IFC model:', error);
            //   } else {
            //     const threeModel = this.reconstructThreeObject(model);
            //     console.log('preparing...');
            //     world.scene.three.add(threeModel);
            //     console.log('finish...');
            //   }
            // };
          } catch (error) {
            console.error('Error fetching IFC file:', error);
          }

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
          this.world = world;
        }
      }
    }
  }

  reconstructThreeObject(data: any): Object3D {
    const group = new Group();
    group.uuid = data.uuid;
    group.name = data.name;

    data.children.forEach((childData: any) => {
      const child = new Object3D(); // Or a more specific type if needed
      child.uuid = childData.uuid;
      child.name = childData.name;
      // Reconstruct geometry and material if needed
      if (childData.geometry) {
        // Reconstruct geometry
      }
      if (childData.material) {
        // Reconstruct material
      }
      group.add(child);
    });

    return group;
  }

  unmounted() {
    this.world?.dispose();
    const viewport = document.querySelector('bim-viewport');
    if (viewport && this.resizeHandler) {
      viewport.removeEventListener('resize', this.resizeHandler);
    }
  }
}

export default toNative(AppIFCViewer);
</script>

<style scoped>
.h-full {
  height: 100%;
}
</style>
