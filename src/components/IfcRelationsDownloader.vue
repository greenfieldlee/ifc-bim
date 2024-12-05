<template>
    <div class="container">
        <div class="card">
            <h1>IFC Relations Downloader</h1>
            <div class="file-input">
                <label for="ifcFile" class="file-label">
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
                        stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                        <polyline points="17 8 12 3 7 8"></polyline>
                        <line x1="12" y1="3" x2="12" y2="15"></line>
                    </svg>
                    <span>Click to upload or drag and drop</span>
                    <span class="file-type">IFC file</span>
                </label>
                <input id="ifcFile" type="file" @change="handleFileSelect" accept=".ifc" />
            </div>
            <button @click="loadAndIndexModel" :disabled="!selectedFile" class="btn btn-primary">
                Load and Index Model
            </button>
            <button @click="downloadAllRelations" :disabled="!modelLoaded" class="btn btn-success">
                Download All Relations
            </button>
            <button @click="downloadModelRelations" :disabled="!modelLoaded" class="btn btn-info">
                Download Model Relations
            </button>
        </div>
    </div>
</template>

<script setup>
import { ref } from 'vue'
import * as OBC from '@thatopen/components'
import * as WEBIFC from "web-ifc";

const components = new OBC.Components()
const ifcLoader = components.get(OBC.IfcLoader)
const indexer = components.get(OBC.IfcRelationsIndexer)

const model = ref(null)
const selectedFile = ref(null)
const modelLoaded = ref(false)

const handleFileSelect = (event) => {
    selectedFile.value = event.target.files[0]
}

const downloadJSON = (json, name) => {
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = name
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
}

const downloadAllRelations = () => {
    const allRelationsJSON = indexer.serializeAllRelations()
    downloadJSON(allRelationsJSON, 'all-relations.json')
}

const downloadModelRelations = () => {
    if (model.value) {
        const modelRelationsJSON = indexer.serializeModelRelations(model.value)
        downloadJSON(modelRelationsJSON, 'model-relations.json')
    } else {
        alert('No model loaded')
    }
}

const loadAndIndexModel = async () => {
    if (!selectedFile.value) {
        alert('Please select an IFC file first')
        return
    }

    try {
        await ifcLoader.setup()
        const buffer = await selectedFile.value.arrayBuffer()
        const modelData = await ifcLoader.load(new Uint8Array(buffer))
        model.value = modelData
        await indexer.process(model.value)
        modelLoaded.value = true
        alert('Model loaded and indexed successfully');

        console.log('modelData', modelData)

        const walls = await modelData.getAllPropertiesOfType(WEBIFC.IFCBUILDING);
        console.log('walls', walls)

        const buildingStorey = indexer.getEntityRelations(
            model,
            WEBIFC.IFCBUILDING,
            "ContainsElements",
        );

        console.log('buildingStorey', buildingStorey)

    } catch (error) {
        console.error('Error loading or indexing model:', error)
        alert('Error loading or indexing model. Please check the console for details.')
    }
}
</script>

<style scoped>
.container {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    background-color: #f0f0f0;
    padding: 20px;
}

.card {
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    padding: 24px;
    width: 100%;
    max-width: 400px;
}

h1 {
    font-size: 24px;
    font-weight: bold;
    text-align: center;
    margin-bottom: 24px;
    color: #333;
}

.file-input {
    margin-bottom: 16px;
}

.file-label {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    border: 2px dashed #ccc;
    border-radius: 8px;
    padding: 20px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.file-label:hover {
    background-color: #f8f8f8;
}

.file-label svg {
    margin-bottom: 8px;
    color: #666;
}

.file-label span {
    font-size: 14px;
    color: #666;
}

.file-label .file-type {
    font-size: 12px;
    color: #999;
}

input[type="file"] {
    display: none;
}

.btn {
    display: block;
    width: 100%;
    padding: 10px;
    margin-bottom: 12px;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
    transition: background-color 0.3s;
}

.btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.btn-primary {
    background-color: #3498db;
    color: white;
}

.btn-primary:hover:not(:disabled) {
    background-color: #2980b9;
}

.btn-success {
    background-color: #2ecc71;
    color: white;
}

.btn-success:hover:not(:disabled) {
    background-color: #27ae60;
}

.btn-info {
    background-color: #9b59b6;
    color: white;
}

.btn-info:hover:not(:disabled) {
    background-color: #8e44ad;
}
</style>