<script setup>
import { ref, computed, watch } from 'vue'

const fileName = ref('')
const fileSize = ref(0) // in bytes
const fromName = ref('Acme')
const subject = ref('')
const body = ref('')
const recipientEmail = ref('')
const emailCode = ref('')
const ccEmail = ref('')
const bccEmail = ref('')
const statusMessage = ref('')
const isEncoding = ref(false)
const isSending = ref(false)
const showAdvanced = ref(false)

const MAX_SIZE_MB = 40
const MAX_SIZE_BYTES = MAX_SIZE_MB * 1024 * 1024

const isFileSizeValid = computed(() => fileSize.value <= MAX_SIZE_BYTES)
const isFormValid = computed(() => {
  return recipientEmail.value && 
         /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(recipientEmail.value) &&
         emailCode.value &&
         isFileSizeValid.value &&
         !isEncoding.value &&
         !isSending.value
})

const formatSize = (bytes) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
    if (file.size > MAX_SIZE_BYTES) {
      statusMessage.value = `文件过大！最大限制为 ${MAX_SIZE_MB}MB`
      fileName.value = ''
      fileSize.value = 0
      return
    }
    fileName.value = file.name
    fileSize.value = file.size
    statusMessage.value = ''
    
    // Set default subject if empty
    if (!subject.value || subject.value.startsWith('文件传输 -')) {
      subject.value = `文件传输 - ${file.name}`
    }
  }
}

const removeFile = () => {
  fileName.value = ''
  fileSize.value = 0
  const fileInput = document.getElementById('relay-file-upload')
  if (fileInput) fileInput.value = ''
  if (subject.value.startsWith('文件传输 -')) {
    subject.value = ''
  }
}

const sendEmail = async () => {
  if (!isFormValid.value) return

  const fileInput = document.getElementById('relay-file-upload')
  const file = fileInput?.files?.[0]

  try {
    let base64String = null
    if (file) {
      isEncoding.value = true
      statusMessage.value = '正在编码文件...'
      const base64Content = await fileToBase64(file)
      base64String = base64Content.split(',')[1]
      isEncoding.value = false
    }
    
    isSending.value = true
    statusMessage.value = '正在通过服务器发送邮件...'

    // Call real Django backend API
    const response = await fetch('http://119.23.182.252:8000/api/email_relay/send/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        email_code: emailCode.value,
        from_name: fromName.value || '智能助理',
        to: recipientEmail.value,
        subject: subject.value || '无主题邮件',
        body: body.value,
        filename: fileName.value || null,
        content: base64String,
        cc: ccEmail.value ? ccEmail.value.split(',').map(e => e.trim()) : null,
        bcc: bccEmail.value ? bccEmail.value.split(',').map(e => e.trim()) : null
      })
    })

    const result = await response.json()

    if (response.ok) {
      statusMessage.value = '邮件发送成功！'
      // Clear form
      recipientEmail.value = ''
      emailCode.value = ''
      fileName.value = ''
      fileSize.value = 0
      subject.value = ''
      body.value = ''
      ccEmail.value = ''
      bccEmail.value = ''
      if (fileInput) fileInput.value = ''
    } else {
      throw new Error(result.error || '服务器响应错误')
    }
  } catch (error) {
    console.error(error)
    statusMessage.value = '发送失败: ' + error.message
  } finally {
    isEncoding.value = false
    isSending.value = false
  }
}

const fileToBase64 = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result)
    reader.onerror = (error) => reject(error)
    reader.readAsDataURL(file)
  })
}
</script>

<template>
  <div class="card email-form-card">
    <h1 class="title">邮箱代发助手</h1>
    
    <div class="form-grid">
      <div class="input-group">
        <label>发件人姓名 (可选)</label>
        <input type="text" v-model="fromName" placeholder="例如: 智能助理" class="text-input" />
      </div>

      <div class="input-group">
        <label>收件人邮箱 <span class="required">*</span></label>
        <input type="email" v-model="recipientEmail" placeholder="请输入收件人邮箱" class="text-input" />
      </div>

      <div class="input-group full-width">
        <label>邮件授权码 <span class="required">*</span></label>
        <input type="text" v-model="emailCode" placeholder="请输入管理员提供的授权码" class="text-input" />
      </div>

      <div class="input-group full-width">
        <label>邮件主题 (可选)</label>
        <input type="text" v-model="subject" placeholder="无主题邮件" class="text-input" />
      </div>

      <div class="input-group full-width">
        <label>邮件正文 (可选)</label>
        <textarea v-model="body" placeholder="请输入邮件内容..." class="text-input textarea" rows="3"></textarea>
      </div>

      <div class="input-group full-width">
        <label>附件选择 (最大 {{ MAX_SIZE_MB }}MB, 可选)</label>
        <div v-if="fileName" class="file-preview-box">
          <div class="file-info">
            <span class="file-name">{{ fileName }}</span>
            <span class="file-size">{{ formatSize(fileSize) }}</span>
          </div>
          <button @click="removeFile" class="remove-file-btn" title="删除附件">移除文件</button>
        </div>
        <label v-else for="relay-file-upload" class="upload-area" :class="{ error: !isFileSizeValid && fileName }">
          <span class="placeholder-text">点击或拖拽附件上传</span>
          <input id="relay-file-upload" type="file" @change="handleFileUpload" hidden />
        </label>
        <div v-if="!isFileSizeValid && fileName" class="error-tip">文件超过 40MB 限制</div>
      </div>

      <div class="advanced-toggle" @click="showAdvanced = !showAdvanced">
        <span>高级选项 (抄送/密送)</span>
        <span class="icon" :class="{ rotated: showAdvanced }">▼</span>
      </div>

      <Transition name="expand">
        <div v-if="showAdvanced" class="advanced-fields full-width">
          <div class="input-group">
            <label>抄送 (CC)</label>
            <input type="text" v-model="ccEmail" placeholder="多个邮箱用逗号分隔" class="text-input" />
          </div>
          <div class="input-group">
            <label>密送 (BCC)</label>
            <input type="text" v-model="bccEmail" placeholder="多个邮箱用逗号分隔" class="text-input" />
          </div>
        </div>
      </Transition>
    </div>

    <button 
      @click="sendEmail" 
      :disabled="!isFormValid"
      class="send-button"
      :class="{ 'loading': isEncoding || isSending }"
    >
      <span v-if="isEncoding">正在编码...</span>
      <span v-else-if="isSending">正在发送...</span>
      <span v-else>立即发送邮件</span>
    </button>

    <div v-if="statusMessage" class="status-msg" :class="{ 'success': statusMessage.includes('成功'), 'error': statusMessage.includes('失败') || statusMessage.includes('大') }">
      {{ statusMessage }}
    </div>

    <div class="footer-hint">
      提示: 附件将以 Base64 方式加密通过 Resend API 传输
    </div>
  </div>
</template>

<style scoped>
.email-form-card {
  max-width: 500px;
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.full-width {
  grid-column: span 2;
}

.placeholder-text {
  color: #9ca3af;
}

.upload-area {
  height: 90px;
  background-color: #f9fafb;
  border: 1px dashed #d1d5db;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: center;
  padding: 12px;
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

.upload-area.error {
  border-color: #ef4444;
  background-color: #fef2f2;
}

.file-preview-box {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f0f7ff;
  border: 1px solid #bfdbfe;
  border-radius: 12px;
  padding: 12px 16px;
}

.file-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.remove-file-btn {
  background: #fee2e2;
  color: #ef4444;
  border: none;
  border-radius: 8px;
  padding: 6px 12px;
  font-size: 11px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
}

.remove-file-btn:hover {
  background: #fecaca;
  color: #dc2626;
}

.file-name {
  font-weight: 600;
  font-size: 13px;
  color: #111827;
  max-width: 240px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-size {
  font-size: 11px;
  color: #6b7280;
}

.text-input {
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 10px 14px;
  color: #111827;
  font-size: 13px;
  outline: none;
  transition: all 0.2s ease;
  width: 100%;
  box-sizing: border-box;
}

.required {
  color: #ef4444;
  margin-left: 2px;
}

.text-input:focus {
  border-color: #2563eb;
  background: #ffffff;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.08);
}

.textarea {
  resize: vertical;
  min-height: 80px;
}

.advanced-toggle {
  grid-column: span 2;
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: #2563eb;
  cursor: pointer;
  user-select: none;
  font-weight: 600;
}

.advanced-toggle .icon {
  font-size: 10px;
  transition: transform 0.2s ease;
}

.advanced-toggle .icon.rotated {
  transform: rotate(180deg);
}

.advanced-fields {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  overflow: hidden;
}

.send-button {
  background: #2563eb;
  border: none;
  border-radius: 12px;
  padding: 14px;
  color: white;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  margin-top: 8px;
  box-shadow: 0 4px 6px -1px rgba(37, 99, 235, 0.2);
}

.send-button:hover:not(:disabled) {
  background: #1d4ed8;
  transform: translateY(-1px);
  box-shadow: 0 6px 15px rgba(37, 99, 235, 0.25);
}

.send-button:disabled {
  background: #93c5fd;
  cursor: not-allowed;
  box-shadow: none;
}

.status-msg {
  text-align: center;
  font-size: 13px;
  padding: 12px;
  border-radius: 10px;
  font-weight: 600;
}

.footer-hint {
  font-size: 10px;
  color: #9ca3af;
  text-align: center;
  margin-top: -8px;
}

/* Animations */
.expand-enter-active, .expand-leave-active {
  transition: all 0.3s ease;
  max-height: 200px;
}
.expand-enter-from, .expand-leave-to {
  max-height: 0;
  opacity: 0;
  margin-top: -16px;
}
</style>
