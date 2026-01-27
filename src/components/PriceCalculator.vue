<script setup>
import { ref, computed } from 'vue'

const fileName = ref('')
const fileSize = ref(0) // in bytes
const retentionDays = ref(2)

const fileSizeGB = computed(() => {
  return fileSize.value / (1024 * 1024 * 1024)
})

const totalPrice = computed(() => {
  const basePrice = 2
  const baseGB = 2
  const baseDays = 2

  // Every additional GB adds 1 yuan
  const extraSizeCost = Math.max(0, Math.ceil(fileSizeGB.value) - baseGB) * 1
  
  // Every additional day adds 1 yuan
  const extraTimeCost = Math.max(0, retentionDays.value - baseDays) * 1

  return basePrice + extraSizeCost + extraTimeCost
})

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
    fileName.value = file.name
    fileSize.value = file.size
  }
}

const formatSize = (bytes) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}
</script>

<template>
  <div class="card">
    <h1 class="title">存储价格计算</h1>
    
    <div class="input-group">
      <label>文件选择</label>
      <label for="file-upload" class="upload-area" :class="{ active: fileName }">
        <span v-if="!fileName" class="placeholder-text">点击或拖拽文件上传</span>
        <div v-else class="file-info">
          <span class="file-name">{{ fileName }}</span>
          <span class="file-size">{{ formatSize(fileSize) }}</span>
        </div>
        <input id="file-upload" type="file" @change="handleFileUpload" hidden />
      </label>
    </div>

    <div class="input-group">
      <label>保留时间 ({{ retentionDays }} 天)</label>
      <div class="range-container">
        <input 
          type="range" 
          v-model.number="retentionDays" 
          min="1" 
          max="30" 
          class="slider"
        />
        <input 
          type="number" 
          v-model.number="retentionDays" 
          min="1" 
          max="365"
          class="days-input" 
        />
      </div>
    </div>

    <div class="result-area">
      <div class="price-label">预计费用</div>
      <div class="price-value">
        <span class="currency">¥</span>
        <span class="amount">{{ totalPrice.toFixed(2) }}</span>
      </div>
    </div>

    <div class="details">
      <div class="detail-item">
        <span class="label">超出容量 (2GB外):</span>
        <span class="value">+¥{{ Math.max(0, Math.ceil(fileSizeGB) - 2).toFixed(2) }}</span>
      </div>
      <div class="detail-item">
        <span class="label">超出天数 (2天外):</span>
        <span class="value">+¥{{ Math.max(0, retentionDays - 2).toFixed(2) }}</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.placeholder-text {
  color: #9ca3af;
}

.upload-area {
  height: 100px;
  background-color: #f9fafb;
  border: 1px dashed #d1d5db;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: center;
  padding: 16px;
}

.upload-area:hover {
  border-color: #2563eb;
  background-color: #eff6ff;
}

.upload-area.active {
  border-style: solid;
  border-color: #2563eb;
  background-color: #eff6ff;
}

.file-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.file-name {
  font-weight: 600;
  font-size: 14px;
  color: #111827;
  max-width: 280px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-size {
  font-size: 12px;
  color: #6b7280;
}

.range-container {
  display: flex;
  align-items: center;
  gap: 12px;
}

.slider {
  flex: 1;
  height: 6px;
  appearance: none;
  background: #e5e7eb;
  border-radius: 3px;
  outline: none;
}

.slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 4px solid #ffffff;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.days-input {
  width: 60px;
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 8px;
  color: #111827;
  font-size: 14px;
  font-weight: 600;
  text-align: center;
}

.result-area {
  background: #eff6ff;
  border-radius: 16px;
  padding: 24px;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.price-label {
  font-size: 13px;
  font-weight: 600;
  color: #1d4ed8;
}

.price-value {
  display: flex;
  align-items: baseline;
  justify-content: center;
  gap: 2px;
  color: #2563eb;
}

.currency {
  font-size: 20px;
  font-weight: 700;
}

.amount {
  font-size: 40px;
  font-weight: 800;
  letter-spacing: -0.5px;
}

.details {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 13px;
}

.detail-item .label {
  color: #6b7280;
}

.detail-item .value {
  font-weight: 600;
  color: #374151;
}
</style>
