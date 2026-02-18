<template>
  <div class="app">
    <header class="app-header">
      <h1>🖥️ ESP32 LED矩阵屏控制器</h1>
      <p class="subtitle">128×32 HUB75 图片上传工具</p>
    </header>
    <main>
      <ImageUploader />
    </main>
  </div>
</template>

<script setup>
import ImageUploader from './components/ImageUploader.vue'
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
  background: #0f0f1a;
  color: #e0e0e0;
  min-height: 100vh;
}

.app {
  max-width: 720px;
  margin: 0 auto;
  padding: 24px 16px;
}

.app-header {
  text-align: center;
  margin-bottom: 32px;
  padding-bottom: 20px;
  border-bottom: 1px solid #2a2a3e;
}

.app-header h1 {
  font-size: 24px;
  font-weight: 600;
  color: #ffffff;
  margin-bottom: 6px;
}

.subtitle {
  font-size: 14px;
  color: #888;
}
</style>
