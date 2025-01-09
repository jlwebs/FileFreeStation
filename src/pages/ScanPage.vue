<template>
  <div class="scan-container">
    <iframe 
      src="/scan.html" 
      class="w-full h-full border-none"
      @load="onIframeLoad"
    ></iframe>
    <div v-if="scanResult" class="scan-result">
      <img :src="scanResult" alt="Scan result" />
      <button @click="saveScan">Save Scan</button>
    </div>
  </div>
</template>

<script lang="ts">
import { ref } from 'vue';
import { PutFile } from '../api/file';
import useFileStore from '../store/file';

export default {
  name: 'ScanPage',
  setup() {
    const fileStore = useFileStore();
    const scanResult = ref<string | null>(null);

    const onIframeLoad = () => {
      // Listen for messages from iframe
      window.addEventListener('message', (event) => {
        if (event.data.type === 'scan-result') {
          scanResult.value = event.data.image;
        }
      });
    };

    const saveScan = async () => {
      if (!scanResult.value) return;
      
      const filename = `scan-${Date.now()}.png`;
      try {
        await PutFile(filename, scanResult.value, fileStore.visibility, 'image');
        alert('Scan saved successfully!');
      } catch (error) {
        console.error('Failed to save scan:', error);
        alert('Failed to save scan');
      }
    };

    return {
      scanResult,
      onIframeLoad,
      saveScan
    };
  }
}
</script>

<style scoped>
.scan-container {
  position: relative;
  width: 100%;
  height: 100vh;
}

.scan-result {
  position: absolute;
  bottom: 20px;
  right: 20px;
  background: white;
  padding: 10px;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.scan-result img {
  max-width: 200px;
  max-height: 200px;
  display: block;
  margin-bottom: 10px;
}

.scan-result button {
  background: #007bff;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
</style>
