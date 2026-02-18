<template>
  <div class="uploader">
    <!-- 设备连接 -->
    <section class="card">
      <h2>📡 设备连接</h2>
      <div class="connection-row">
        <input
          v-model="deviceIP"
          type="text"
          placeholder="输入ESP32 IP地址，例如 192.168.1.100"
          class="input-ip"
          @keyup.enter="checkConnection"
        />
        <button @click="checkConnection" :disabled="!deviceIP || checking" class="btn btn-check">
          {{ checking ? '检测中...' : '检测连接' }}
        </button>
      </div>
      <p class="hint">提示：也可尝试 <code>esp32-led.local</code>（需mDNS支持）</p>
      <div v-if="deviceInfo" class="device-info">
        <span class="badge badge-ok">✓ 已连接</span>
        <span>分辨率: {{ deviceInfo.width }}×{{ deviceInfo.height }}</span>
        <span>信号: {{ deviceInfo.rssi }} dBm</span>
      </div>
      <div v-if="connectionError" class="device-info">
        <span class="badge badge-err">✗ 连接失败</span>
        <span>{{ connectionError }}</span>
      </div>
    </section>

    <!-- 图片选择与预览 -->
    <section class="card">
      <h2>🖼️ 图片上传</h2>

      <div
        class="drop-zone"
        :class="{ 'drop-zone-active': isDragging }"
        @click="triggerFileInput"
        @dragover.prevent="isDragging = true"
        @dragleave.prevent="isDragging = false"
        @drop.prevent="onDrop"
      >
        <input
          ref="fileInput"
          type="file"
          accept="image/png"
          @change="onFileSelect"
          style="display: none"
        />
        <div v-if="!previewUrl" class="drop-placeholder">
          <span class="drop-icon">📁</span>
          <p>点击选择或拖放 PNG 图片</p>
          <p class="hint">要求尺寸：128×32 像素</p>
        </div>
        <div v-else class="preview-wrapper">
          <!-- 原始图片预览 -->
          <div class="preview-label">原图预览</div>
          <img :src="previewUrl" alt="原图预览" class="preview-img" />
          <!-- LED模拟预览 (放大像素化) -->
          <div class="preview-label" style="margin-top: 16px">LED屏幕模拟 (5x放大)</div>
          <canvas
            ref="ledCanvas"
            width="128"
            height="32"
            class="led-preview"
          ></canvas>
          <div class="image-meta">
            <span>📄 {{ fileName }}</span>
            <span>📐 {{ imageDimensions }}</span>
            <span>💾 {{ fileSize }}</span>
          </div>
        </div>
      </div>

      <!-- 操作按钮 -->
      <div class="actions" v-if="previewUrl">
        <button @click="triggerFileInput" class="btn btn-secondary">重新选择</button>
        <button
          @click="uploadImage"
          :disabled="uploading || !deviceIP || !rgbData"
          class="btn btn-upload"
        >
          {{ uploading ? '上传中...' : '🚀 上传到LED屏幕' }}
        </button>
        <button
          @click="clearScreen"
          :disabled="!deviceIP"
          class="btn btn-secondary"
        >
          清空屏幕
        </button>
      </div>

      <!-- 上传结果 -->
      <div v-if="uploadResult" :class="['result', uploadResult.ok ? 'result-ok' : 'result-err']">
        {{ uploadResult.message }}
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

// ========== 状态 ==========
const deviceIP = ref(localStorage.getItem('esp32_led_ip') || '')
const checking = ref(false)
const deviceInfo = ref(null)
const connectionError = ref('')

const fileInput = ref(null)
const ledCanvas = ref(null)
const previewUrl = ref('')
const fileName = ref('')
const fileSize = ref('')
const imageDimensions = ref('')
const isDragging = ref(false)

const uploading = ref(false)
const uploadResult = ref(null)

let rgbData = null

// 保存IP到localStorage
watch(deviceIP, (val) => {
  if (val) localStorage.setItem('esp32_led_ip', val)
})

// ========== 设备连接 ==========
async function checkConnection() {
  if (!deviceIP.value) return
  checking.value = true
  deviceInfo.value = null
  connectionError.value = ''

  try {
    const res = await fetch(`http://${deviceIP.value}/status`, { signal: AbortSignal.timeout(5000) })
    if (res.ok) {
      deviceInfo.value = await res.json()
    } else {
      connectionError.value = `HTTP ${res.status}`
    }
  } catch (e) {
    connectionError.value = e.name === 'TimeoutError' ? '连接超时' : '无法连接到设备'
  } finally {
    checking.value = false
  }
}

// ========== 文件选择 ==========
function triggerFileInput() {
  fileInput.value?.click()
}

function onFileSelect(e) {
  const file = e.target.files?.[0]
  if (file) processFile(file)
}

function onDrop(e) {
  isDragging.value = false
  const file = e.dataTransfer.files?.[0]
  if (file && file.type === 'image/png') {
    processFile(file)
  }
}

function processFile(file) {
  uploadResult.value = null
  rgbData = null

  fileName.value = file.name
  fileSize.value = (file.size / 1024).toFixed(1) + ' KB'

  if (previewUrl.value) URL.revokeObjectURL(previewUrl.value)
  previewUrl.value = URL.createObjectURL(file)

  const img = new Image()
  img.onload = () => {
    imageDimensions.value = `${img.width}×${img.height}`

    // 绘制到Canvas用于数据提取
    const canvas = ledCanvas.value
    if (!canvas) return
    const ctx = canvas.getContext('2d')
    ctx.clearRect(0, 0, 128, 32)
    ctx.drawImage(img, 0, 0, 128, 32)

    // 提取RGB888数据 (去除Alpha通道)
    const imageData = ctx.getImageData(0, 0, 128, 32)
    const pixelCount = 128 * 32
    const rgb = new Uint8Array(pixelCount * 3)
    for (let i = 0; i < pixelCount; i++) {
      rgb[i * 3]     = imageData.data[i * 4]     // R
      rgb[i * 3 + 1] = imageData.data[i * 4 + 1] // G
      rgb[i * 3 + 2] = imageData.data[i * 4 + 2] // B
    }
    rgbData = rgb
  }
  img.src = previewUrl.value
}

// ========== 上传图片 ==========
async function uploadImage() {
  if (!rgbData || !deviceIP.value) return

  uploading.value = true
  uploadResult.value = null

  try {
    // 将RGB数据转为Base64字符串
    let binary = ''
    const chunkSize = 8192
    for (let i = 0; i < rgbData.length; i += chunkSize) {
      const chunk = rgbData.subarray(i, Math.min(i + chunkSize, rgbData.length))
      binary += String.fromCharCode.apply(null, chunk)
    }
    const base64 = btoa(binary)

    const res = await fetch(`http://${deviceIP.value}/upload`, {
      method: 'POST',
      headers: { 'Content-Type': 'text/plain' },
      body: base64
    })

    const data = await res.json()
    if (res.ok) {
      uploadResult.value = { ok: true, message: '✓ 图片已成功上传到LED屏幕！' }
    } else {
      uploadResult.value = { ok: false, message: `✗ 上传失败: ${data.error || '未知错误'}` }
    }
  } catch (e) {
    uploadResult.value = { ok: false, message: `✗ 网络错误: ${e.message}` }
  } finally {
    uploading.value = false
  }
}

// ========== 清空屏幕 ==========
async function clearScreen() {
  try {
    const res = await fetch(`http://${deviceIP.value}/clear`, { method: 'POST' })
    if (res.ok) {
      uploadResult.value = { ok: true, message: '✓ 屏幕已清空' }
    }
  } catch (e) {
    uploadResult.value = { ok: false, message: `✗ 清空失败: ${e.message}` }
  }
}
</script>

<style scoped>
.uploader {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* 卡片 */
.card {
  background: #1a1a2e;
  border: 1px solid #2a2a3e;
  border-radius: 12px;
  padding: 20px;
}

.card h2 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 14px;
  color: #fff;
}

/* 连接行 */
.connection-row {
  display: flex;
  gap: 10px;
  margin-bottom: 8px;
}

.input-ip {
  flex: 1;
  padding: 10px 14px;
  background: #12121f;
  border: 1px solid #3a3a50;
  border-radius: 8px;
  color: #e0e0e0;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.input-ip:focus {
  border-color: #6366f1;
}

.hint {
  font-size: 12px;
  color: #666;
  margin-top: 4px;
}

.hint code {
  background: #22223a;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 12px;
  color: #a5b4fc;
}

/* 设备信息 */
.device-info {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 12px;
  font-size: 13px;
  color: #aaa;
}

.badge {
  padding: 3px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.badge-ok {
  background: #064e3b;
  color: #34d399;
}

.badge-err {
  background: #4c1d1d;
  color: #f87171;
}

/* 按钮 */
.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  color: #fff;
}

.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.btn-check {
  background: #3b3b5c;
}

.btn-check:hover:not(:disabled) {
  background: #4b4b6c;
}

.btn-upload {
  background: #4f46e5;
  flex: 1;
}

.btn-upload:hover:not(:disabled) {
  background: #6366f1;
}

.btn-secondary {
  background: #2a2a3e;
}

.btn-secondary:hover:not(:disabled) {
  background: #3a3a50;
}

/* 拖放区域 */
.drop-zone {
  border: 2px dashed #3a3a50;
  border-radius: 10px;
  padding: 24px;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
  min-height: 120px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.drop-zone:hover,
.drop-zone-active {
  border-color: #6366f1;
  background: #1e1e35;
}

.drop-placeholder {
  color: #888;
}

.drop-icon {
  font-size: 36px;
  display: block;
  margin-bottom: 8px;
}

/* 预览 */
.preview-wrapper {
  width: 100%;
}

.preview-label {
  font-size: 12px;
  color: #888;
  margin-bottom: 6px;
  text-align: left;
}

.preview-img {
  width: 100%;
  max-width: 640px;
  height: auto;
  border-radius: 6px;
  border: 1px solid #2a2a3e;
  image-rendering: pixelated;
  background: #000;
}

.led-preview {
  width: 100%;
  max-width: 640px;
  height: auto;
  border-radius: 6px;
  border: 1px solid #2a2a3e;
  image-rendering: pixelated;
  background: #000;
}

.image-meta {
  display: flex;
  gap: 16px;
  margin-top: 10px;
  font-size: 12px;
  color: #888;
  text-align: left;
}

/* 操作按钮 */
.actions {
  display: flex;
  gap: 10px;
  margin-top: 16px;
}

/* 结果提示 */
.result {
  margin-top: 14px;
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
}

.result-ok {
  background: #064e3b;
  color: #34d399;
  border: 1px solid #065f46;
}

.result-err {
  background: #4c1d1d;
  color: #f87171;
  border: 1px solid #7f1d1d;
}
</style>
