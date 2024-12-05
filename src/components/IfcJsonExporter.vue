<template>
    <div class="container">
      <h1>IFC to JSON Exporter</h1>
      <div class="file-input">
        <label for="ifcFile" class="file-label">
          <span>Click to upload or drag and drop</span>
          <span class="file-type">IFC file</span>
        </label>
        <input id="ifcFile" type="file" @change="handleFileSelect" accept=".ifc" />
      </div>
      <bim-button 
        label="Export properties JSON" 
        @click="exportToJson"
        :disabled="!ifcLoaded">
      </bim-button>
      <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted } from 'vue';
  import * as BUI from "@thatopen/ui";
  import * as WEBIFC from "web-ifc";
  import * as OBC from "@thatopen/components";
  
  const components = new OBC.Components();
  const exporter = components.get(OBC.IfcJsonExporter);
  
  const ifcBuffer = ref(null);
  const ifcLoaded = ref(false);
  const errorMessage = ref('');
  let webIfc;
  let modelID;
  
  onMounted(async () => {
    BUI.Manager.init();
    
    webIfc = new WEBIFC.IfcAPI();
    webIfc.SetWasmPath("https://unpkg.com/web-ifc@0.0.57/", true);
    await webIfc.Init();
  });
  
  const handleFileSelect = async (event) => {
    const file = event.target.files[0];
    if (file) {
      try {
        const arrayBuffer = await file.arrayBuffer();
        ifcBuffer.value = new Uint8Array(arrayBuffer);
        modelID = webIfc.OpenModel(ifcBuffer.value);
        ifcLoaded.value = true;
        errorMessage.value = '';
      } catch (error) {
        console.error('Error reading file:', error);
        errorMessage.value = 'Error reading file. Please try again.';
      }
    }
  };
  
  const exportToJson = async () => {
    if (!ifcLoaded.value) {
      errorMessage.value = 'Please load an IFC file first.';
      return;
    }
  
    try {
      const exported = await exporter.export(webIfc, modelID);
      const serialized = JSON.stringify(exported);
      const file = new File([new Blob([serialized])], "properties.json");
      const url = URL.createObjectURL(file);
      const link = document.createElement("a");
      link.download = "properties.json";
      link.href = url;
      link.click();
      URL.revokeObjectURL(url);
      link.remove();
      
      errorMessage.value = '';
    } catch (error) {
      console.error('Error exporting to JSON:', error);
      errorMessage.value = 'Error exporting to JSON. Please check the console for details.';
    }
  };
  </script>
  
  <style scoped>
  .container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
    font-family: Arial, sans-serif;
  }
  
  h1 {
    text-align: center;
    color: #333;
  }
  
  .file-input {
    margin-bottom: 20px;
  }
  
  .file-label {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    border: 2px dashed #ccc;
    border-radius: 4px;
    padding: 20px;
    cursor: pointer;
    transition: background-color 0.3s;
  }
  
  .file-label:hover {
    background-color: #f8f8f8;
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
  
  .error-message {
    margin-top: 10px;
    color: #dc3545;
    text-align: center;
  }
  </style>
  
  