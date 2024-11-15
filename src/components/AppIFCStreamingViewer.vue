<template>
    <div>
        <div id="ifc-container" class="h-full"></div>
    </div>
</template>

<script lang="ts">
import { Component, Prop, toNative, Vue } from 'vue-facing-decorator';
import * as OBC from '@thatopen/components';
import * as OBCF from '@thatopen/components-front';
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';
import { AmbientLight, DirectionalLight } from 'three';

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
class AppIFCStreamingViewer extends Vue {
    @Prop({ default: '', type: String }) geometryURL!: string;
    @Prop({ default: '', type: String }) propertiesURL!: string;
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

            // Set up IFC Streamer
            const loader = components.get(OBCF.IfcStreamer);
            loader.world = world;
            loader.url = "http://localhost:8080/"; // Set this to your backend URL if needed

            // Load the model if URLs are provided
            if (this.geometryURL && this.propertiesURL) {

                try {
                    const rawGeometryData = await fetch(this.geometryURL);
                    if (!rawGeometryData.ok) {
                        throw new Error(`Failed to fetch geometry data: ${rawGeometryData.statusText}`);
                    }
                    const geometryData = await rawGeometryData.json();

                    const rawPropertiesData = await fetch(this.propertiesURL);
                    if (!rawPropertiesData.ok) {
                        throw new Error(`Failed to fetch properties data: ${rawPropertiesData.statusText}`);
                    }
                    const propertiesData = await rawPropertiesData.json();

                    console.log('geometryData', geometryData);
                    console.log('propertiesData', propertiesData);
                    const model = await loader.load(geometryData, true, propertiesData);
                    console.log("Model loaded:", model);

                    // Update classifications after model is loaded
                    classifier.byEntity(model);
                    await classifier.byPredefinedType(model);
                    const classifications = [
                        { system: 'entities', label: 'Entities' },
                        { system: 'predefinedTypes', label: 'Predefined Types' }
                    ];
                    updateClassificationsTree({ classifications });
                } catch (error) {
                    console.error('Error loading IFC model:', error);
                }

            }

            world.camera.controls.addEventListener("sleep", () => {
                loader.culler.needsUpdate = true;
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

    unmounted() {
        this.world?.dispose();
        const viewport = document.querySelector('bim-viewport');
        if (viewport && this.resizeHandler) {
            viewport.removeEventListener('resize', this.resizeHandler);
        }
    }
}

export default toNative(AppIFCStreamingViewer);
</script>

<style scoped>
.h-full {
    height: 100%;
}
</style>