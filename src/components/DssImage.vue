<script setup>
import { ref, onMounted } from "vue";

const ra = ref(10.67);
const dec = ref(41.26);
const exposureTime = ref(1);
const pixels = ref(500);
const canvasRef = ref(null);
const loading = ref(false);
const errorMsg = ref("");

const getDSSImageWithNoise = async () => {
  loading.value = true;
  errorMsg.value = "";

  try {
    const proxyUrl = `https://minidsserver-942fd28e2022.herokuapp.com/proxy?ra=${ra.value}&dec=${dec.value}&fov=0.5&pixels=${pixels.value}`;
    const response = await fetch(proxyUrl);
    
    if (!response.ok) throw new Error("Failed to retrieve image from proxy");

    const blob = await response.blob();
    const img = new Image();
    img.src = URL.createObjectURL(blob);
    
    img.onload = () => {
      const canvas = canvasRef.value;
      const ctx = canvas.getContext("2d");
      ctx.drawImage(img, 0, 0, pixels.value, pixels.value);

      let imageData = ctx.getImageData(0, 0, pixels.value, pixels.value);
      let data = imageData.data;

      let noiseLevel = 1 - Math.min(Math.max(exposureTime.value / 300, 0), 1);
      for (let i = 0; i < data.length; i += 4) {
        let noise = (Math.random() - 0.5) * noiseLevel * 255;
        let intensity = data[i] + noise;
        intensity = Math.max(0, Math.min(255, intensity));
        data[i] = data[i + 1] = data[i + 2] = intensity;
      }

      ctx.putImageData(imageData, 0, 0);
      loading.value = false;
    };
  } catch (error) {
    errorMsg.value = error.message;
    loading.value = false;
  }
};

onMounted(getDSSImageWithNoise);
</script>

<template>
  <div class="container">
    <h1>DSS Image with Simulated Noise</h1>

    <div class="controls">
      <label>RA: <input type="number" v-model="ra" /></label>
      <label>Dec: <input type="number" v-model="dec" /></label>
      <label>Exposure Time: <input type="number" v-model="exposureTime" /></label>
      <button @click="getDSSImageWithNoise" :disabled="loading">Load Image</button>
    </div>

    <canvas ref="canvasRef" :width="pixels" :height="pixels"></canvas>

    <p v-if="loading">Loading...</p>
    <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
  </div>
</template>

<style>
.container {
  text-align: center;
  margin: 20px;
}

.controls {
  margin-bottom: 10px;
}

canvas {
  border: 1px solid #ccc;
  margin-top: 10px;
}

.error {
  color: red;
}
</style>
