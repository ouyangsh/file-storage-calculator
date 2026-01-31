<script setup>
import { ref, computed } from 'vue'

const fileName = ref('')
const fileSize = ref(0) // in bytes
const retentionDays = ref(2)
const fileObject = ref(null)

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
    fileObject.value = file
  }
}

const formatSize = (bytes) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

// --- Order & Upload Logic ---
const API_BASE = window.location.protocol === 'https:' 
  ? 'https://119.23.182.252.sslip.io/api' 
  : 'http://119.23.182.252:8000/api'

const showSuccessDialog = ref(false)
const queryCode = ref('')
const confirmCode = ref('')
const uploadUrl = ref('')
const downloadUrl = ref('')
const isUploading = ref(false)
const uploadProgress = ref(0)
const isOrderCreated = ref(false) // Track if order is created

const confirmOrder = async () => {
  if (!fileName.value) {
    alert('请先选择文件')
    return
  }
  
  try {
    const response = await fetch(`${API_BASE}/storage/create_order/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        filename: fileName.value,
        filesize: fileSize.value,
        retention_days: retentionDays.value
      })
    })
    
    const data = await response.json()
    if (response.ok) {
      queryCode.value = data.query_code
      isOrderCreated.value = true
    } else {
      alert('创建订单失败: ' + (data.error || 'Unknown error'))
    }
  } catch (e) {
    alert('网络错误: ' + e.message)
  }
}

const verifyAndUpload = async () => {
  if (!confirmCode.value) {
    alert('请输入确认码')
    return
  }
  
  if (!fileName.value || !fileObject.value) {
    alert('请先选择文件')
    return
  }

  try {
    // 1. Verify
    const verifyRes = await fetch(`${API_BASE}/storage/verify_order/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        query_code: queryCode.value,
        confirm_code: confirmCode.value,
        filename: fileName.value,  // 文件名验证
        filesize: fileSize.value   // 文件大小验证
      })
    })
    const verifyData = await verifyRes.json()
    
    if (!verifyRes.ok) {
      alert('验证失败: ' + (verifyData.error || 'Check code'))
      return
    }


    uploadUrl.value = verifyData.upload_url
    downloadUrl.value = verifyData.download_url

    // 2. Upload to COS with progress tracking
    isUploading.value = true
    uploadProgress.value = 0
    
    // 使用 XMLHttpRequest 来追踪上传进度
    await new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest()
      
      // 监听上传进度
      xhr.upload.addEventListener('progress', (e) => {
        if (e.lengthComputable) {
          uploadProgress.value = Math.round((e.loaded / e.total) * 100)
        }
      })
      
      // 监听上传完成
      xhr.addEventListener('load', () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          showSuccessDialog.value = true
          resolve()
        } else {
          alert('文件上传失败')
          reject(new Error('Upload failed'))
        }
      })
      
      // 监听错误
      xhr.addEventListener('error', () => {
        alert('上传过程中发生网络错误')
        reject(new Error('Network error'))
      })
      
      // 发送请求
      xhr.open('PUT', uploadUrl.value)
      
      // 设置必要的请求头
      if (verifyData.upload_headers && verifyData.upload_headers['x-cos-tagging']) {
        xhr.setRequestHeader('x-cos-tagging', verifyData.upload_headers['x-cos-tagging'])
      }
      
      xhr.send(fileObject.value)
    })
    
  } catch (e) {
    alert('上传过程中发生错误: ' + e.message)
  } finally {
    isUploading.value = false
  }
}

const copyDownloadLink = () => {
  navigator.clipboard.writeText(downloadUrl.value)
  alert('链接已复制')
}

const resetFlow = () => {
  showSuccessDialog.value = false
  fileName.value = ''
  fileSize.value = 0
  fileObject.value = null
  queryCode.value = ''
  confirmCode.value = ''
  isOrderCreated.value = false
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

    <!-- Step 1: Confirm Order -->
    <button v-if="!isOrderCreated" class="confirm-btn" @click="confirmOrder">确认订单 (Confirm Order)</button>
    
    <!-- Step 2: Verification & Upload -->
    <div v-else class="verification-area">
      <div class="code-display">
        <span class="label">查询码 (Query Code):</span>
        <span class="code">{{ queryCode }}</span>
        <p class="hint">请联系管理员获取确认码 / Contact Admin for Code</p>
      </div>

      <div class="input-group">
        <label>确认码 (Confirm Code)</label>
        <input v-model="confirmCode" type="text" class="text-input" placeholder="输入6位确认码" />
      </div>

      <button 
        class="confirm-btn upload-btn" 
        @click="verifyAndUpload" 
        :disabled="!confirmCode || isUploading"
      >
        {{ isUploading ? `上传中 ${uploadProgress}%` : '上传文件 (Upload File)' }}
      </button>
      
      <!-- Upload Progress Bar -->
      <div v-if="isUploading" class="progress-container">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: uploadProgress + '%' }"></div>
        </div>
        <div class="progress-text">{{ uploadProgress }}%</div>
      </div>
    </div>

    <!-- Success Dialog -->
    <div v-if="showSuccessDialog" class="overlay">
      <div class="dialog success">
        <div class="icon">✅</div>
        <h3>上传成功!</h3>
        <p>文件已安全上传至云端。</p>
        <div class="link-box">
          <input readonly :value="downloadUrl" />
          <button @click="copyDownloadLink">复制</button>
        </div>
        <button class="action-btn" @click="resetFlow">完成</button>
      </div>
    </div>

  </div>
</template>

<style scoped>
/* Reusing existing styles */
.placeholder-text { color: #9ca3af; }
.upload-area { height: 100px; background-color: #f9fafb; border: 1px dashed #d1d5db; border-radius: 12px; display: flex; align-items: center; justify-content: center; cursor: pointer; transition: all 0.2s ease; text-align: center; padding: 16px; }
.upload-area:hover { border-color: #2563eb; background-color: #eff6ff; }
.upload-area.active { border-style: solid; border-color: #2563eb; background-color: #eff6ff; }
.file-info { display: flex; flex-direction: column; gap: 2px; }
.file-name { font-weight: 600; font-size: 14px; color: #111827; max-width: 280px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.file-size { font-size: 12px; color: #6b7280; }
.range-container { display: flex; align-items: center; gap: 12px; }
.slider { flex: 1; height: 6px; appearance: none; background: #e5e7eb; border-radius: 3px; outline: none; }
.slider::-webkit-slider-thumb { appearance: none; width: 20px; height: 20px; background: #2563eb; border-radius: 50%; cursor: pointer; border: 4px solid #ffffff; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.days-input { width: 60px; background: #f9fafb; border: 1px solid #e5e7eb; border-radius: 8px; padding: 8px; color: #111827; font-size: 14px; font-weight: 600; text-align: center; }
.result-area { background: #eff6ff; border-radius: 16px; padding: 24px; text-align: center; display: flex; flex-direction: column; gap: 4px; }
.price-label { font-size: 13px; font-weight: 600; color: #1d4ed8; }
.price-value { display: flex; align-items: baseline; justify-content: center; gap: 2px; color: #2563eb; }
.currency { font-size: 20px; font-weight: 700; }
.amount { font-size: 40px; font-weight: 800; letter-spacing: -0.5px; }
.details { display: flex; flex-direction: column; gap: 12px; }
.detail-item { display: flex; justify-content: space-between; align-items: center; font-size: 13px; }
.detail-item .label { color: #6b7280; }
.detail-item .value { font-weight: 600; color: #374151; }

.confirm-btn {
  background-color: #2563eb;
  color: white;
  border: none;
  padding: 12px;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  margin-top: 16px;
  transition: background-color 0.2s;
  width: 100%;
}
.confirm-btn:hover { background-color: #1d4ed8; }
.confirm-btn:disabled { background-color: #9ca3af; cursor: not-allowed; }

.verification-area {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid #e5e7eb;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.code-display {
  background: #f3f4f6;
  padding: 12px;
  border-radius: 8px;
  text-align: center;
}
.code-display .label { font-size: 12px; color: #6b7280; display: block; margin-bottom: 4px; }
.code-display .code { font-size: 24px; font-weight: 700; color: #2563eb; letter-spacing: 2px; }
.code-display .hint { font-size: 12px; color: #9ca3af; margin: 4px 0 0 0; }

.text-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  margin-top: 4px;
}

/* Dialog Styles */
.overlay { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 100; }
.dialog { background: white; padding: 24px; border-radius: 16px; width: 90%; max-width: 400px; box-shadow: 0 20px 25px -5px rgba(0,0,0,0.1); display: flex; flex-direction: column; gap: 16px; text-align: center; }
.dialog h3 { margin: 0; color: #111827; }
.dialog .icon { font-size: 48px; margin-bottom: 8px; }
.link-box { display: flex; gap: 8px; }
.link-box input { flex: 1; border: 1px solid #d1d5db; border-radius: 6px; padding: 8px; font-size: 12px; }
.link-box button { background: #e5e7eb; border: none; padding: 0 12px; border-radius: 6px; cursor: pointer; }
.action-btn { background: #2563eb; color: white; border: none; padding: 10px; border-radius: 8px; font-weight: 600; cursor: pointer; }

/* Progress Bar Styles */
.progress-container {
  margin-top: 16px;
  width: 100%;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background-color: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 8px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #2563eb 0%, #3b82f6 100%);
  transition: width 0.3s ease;
  border-radius: 4px;
}

.progress-text {
  text-align: center;
  font-size: 14px;
  font-weight: 600;
  color: #2563eb;
}
</style>
