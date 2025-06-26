<script setup>
import { ref, reactive, onMounted, computed } from 'vue'

// 响应式数据
const showSettings = ref(false)
const isLoading = ref(false)
const audioUrl = ref('')
const audioBlob = ref(null)
const progress = ref(0)

// 配置数据
const config = reactive({
  apiKey: '',
  groupId: ''
})

// T2A 参数配置
const t2aConfig = reactive({
  voiceId: 'male-qn-qingse',
  speed: 1.0,
  vol: 1.0,
  pitch: 0,
  emotion: 'neutral'
})

// 文本输入
const inputText = ref('')
const maxChars = 5000

// 音色选项
const voiceOptions = [
  { id: 'male-qn-qingse', name: '青年男声' },
  { id: 'female-shaonv', name: '少女' },
  { id: 'male-qingshu', name: '青叔' },
  { id: 'female-tianmei', name: '甜美女声' }
]

// 情感选项
const emotionOptions = [
  { value: 'neutral', label: '中性' },
  { value: 'happy', label: '开心' },
  { value: 'sad', label: '悲伤' },
  { value: 'angry', label: '愤怒' },
  { value: 'fearful', label: '恐惧' },
  { value: 'disgusted', label: '厌恶' },
  { value: 'surprised', label: '惊讶' }
]

// 页面初始化
onMounted(() => {
  loadConfig()
})

// 加载配置
function loadConfig() {
  const savedConfig = localStorage.getItem('minimax-config')
  if (savedConfig) {
    const parsed = JSON.parse(savedConfig)
    config.apiKey = parsed.apiKey || ''
    config.groupId = parsed.groupId || ''
  }
}

// 保存配置
function saveConfig() {
  localStorage.setItem('minimax-config', JSON.stringify({
    apiKey: config.apiKey,
    groupId: config.groupId
  }))
  showSettings.value = false
  // 显示保存成功提示
  const toast = document.createElement('div')
  toast.className = 'toast toast-top toast-end'
  toast.innerHTML = `
    <div class="alert alert-success">
      <span>配置保存成功！</span>
    </div>
  `
  document.body.appendChild(toast)
  setTimeout(() => {
    document.body.removeChild(toast)
  }, 2000)
}

// 开始合成
async function startSynthesis() {
  if (!inputText.value.trim()) {
    alert('请输入要合成的文本')
    return
  }
  
  if (!config.apiKey || !config.groupId) {
    alert('请先配置 API Key 和 Group ID')
    showSettings.value = true
    return
  }

  isLoading.value = true
  progress.value = 0
  audioUrl.value = ''
  audioBlob.value = null

  try {
    const response = await fetch(`https://api.minimaxi.com/v1/t2a_v2?GroupId=${config.groupId}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${config.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: 'speech-02-turbo',
        text: inputText.value,
        stream: true,
        output_format: 'hex',
        stream_options: {
          exclude_aggregated_audio: true
        },
        voice_setting: {
          voice_id: t2aConfig.voiceId,
          speed: t2aConfig.speed,
          vol: t2aConfig.vol,
          pitch: t2aConfig.pitch
        },
        audio_setting: {
          sample_rate: 32000,
          bitrate: 128000,
          format: 'mp3',
          channel: 1
        },
        emotion: t2aConfig.emotion
      })
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    const audioChunks = []
    let buffer = ''
    const processedChunks = new Set() // 用于跟踪已处理的音频块

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      // 将新数据添加到缓冲区
      buffer += decoder.decode(value, { stream: true })
      
      // 按行分割，保留最后一个可能不完整的行
      const lines = buffer.split('\n')
      buffer = lines.pop() || '' // 保留最后一个可能不完整的行

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          try {
            const jsonStr = line.slice(6).trim()
            if (jsonStr && jsonStr !== '{}' && jsonStr !== '[DONE]') {
              const data = JSON.parse(jsonStr)
              if (data.data && data.data.audio) {
                const hexString = data.data.audio
                
                // 使用音频数据的哈希值来检测重复
                const chunkHash = hexString.substring(0, 32) // 使用前32个字符作为简单哈希
                if (!processedChunks.has(chunkHash)) {
                  processedChunks.add(chunkHash)
                  
                  // 将 hex 字符串转换为 Uint8Array
                  const audioData = new Uint8Array(hexString.length / 2)
                  for (let i = 0; i < hexString.length; i += 2) {
                    audioData[i / 2] = parseInt(hexString.substr(i, 2), 16)
                  }
                  audioChunks.push(audioData)
                  console.log(`添加音频块: ${audioChunks.length}, 大小: ${audioData.length}`)
                } else {
                  console.log('检测到重复音频块，已跳过')
                }

                // 更新进度
                if (data.data.status === 2) {
                  progress.value = 100
                } else {
                  progress.value = Math.min(progress.value + 10, 90)
                }
              }

              if (data.base_resp && data.base_resp.status_code !== 0) {
                throw new Error(data.base_resp.status_msg || '合成失败')
              }
            }
          } catch (e) {
            console.warn('解析数据失败:', e, '原始数据:', line)
          }
        }
      }
    }

    // 处理缓冲区中剩余的数据
    if (buffer.trim() && buffer.startsWith('data: ')) {
      try {
        const jsonStr = buffer.slice(6).trim()
        if (jsonStr && jsonStr !== '{}' && jsonStr !== '[DONE]') {
          const data = JSON.parse(jsonStr)
          if (data.data && data.data.audio) {
            const hexString = data.data.audio
            
            // 使用相同的重复检测逻辑
            const chunkHash = hexString.substring(0, 32)
            if (!processedChunks.has(chunkHash)) {
              processedChunks.add(chunkHash)
              
              const audioData = new Uint8Array(hexString.length / 2)
              for (let i = 0; i < hexString.length; i += 2) {
                audioData[i / 2] = parseInt(hexString.substr(i, 2), 16)
              }
              audioChunks.push(audioData)
              console.log(`缓冲区添加音频块: ${audioChunks.length}, 大小: ${audioData.length}`)
            } else {
              console.log('缓冲区检测到重复音频块，已跳过')
            }
          }
        }
      } catch (e) {
        console.warn('解析缓冲区数据失败:', e)
      }
    }

    console.log(`总共处理了 ${audioChunks.length} 个音频块`)

    // 合并所有音频块
    const totalLength = audioChunks.reduce((sum, chunk) => sum + chunk.length, 0)
    const mergedAudio = new Uint8Array(totalLength)
    let offset = 0
    for (const chunk of audioChunks) {
      mergedAudio.set(chunk, offset)
      offset += chunk.length
    }

    // 创建 Blob 和 URL
    audioBlob.value = new Blob([mergedAudio], { type: 'audio/mp3' })
    audioUrl.value = URL.createObjectURL(audioBlob.value)
    progress.value = 100

  } catch (error) {
    console.error('合成失败:', error)
    alert('合成失败: ' + error.message)
  } finally {
    isLoading.value = false
  }
}

// 下载音频
function downloadAudio() {
  if (!audioBlob.value) return

  // 生成文件名
  const textPrefix = inputText.value.trim().substring(0, 5).replace(/[^\u4e00-\u9fa5a-zA-Z0-9]/g, '')
  const now = new Date()
  const timeStr = `${now.getHours().toString().padStart(2, '0')}_${now.getMinutes().toString().padStart(2, '0')}`
  const filename = `${textPrefix}-${timeStr}.mp3`

  // 创建下载链接
  const link = document.createElement('a')
  link.href = audioUrl.value
  link.download = filename
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}

// 字符计数
const charCount = computed(() => inputText.value.length)
const isOverLimit = computed(() => charCount.value > maxChars)
</script>

<template>
  <div class="min-h-screen bg-base-100">
    <!-- 顶部标题栏 -->
    <div class="navbar bg-base-200 shadow-sm">
      <div class="flex-1">
        <h1 class="text-xl font-bold">MiniMax T2A Web UI</h1>
      </div>
      <div class="flex-none">
        <button class="btn btn-ghost btn-circle" @click="showSettings = true">
          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z" />
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
          </svg>
        </button>
      </div>
    </div>

    <!-- 主要内容区域 -->
    <div class="container mx-auto p-4">
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- 左侧配置面板 -->
        <div class="lg:col-span-1">
          <div class="card bg-base-200 shadow-sm">
            <div class="card-body">
              <h2 class="card-title text-lg mb-4">语音配置</h2>
              
              <!-- 音色选择 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text">音色</span>
                </label>
                <select class="select select-bordered w-full" v-model="t2aConfig.voiceId">
                  <option v-for="voice in voiceOptions" :key="voice.id" :value="voice.id">
                    {{ voice.name }}
                  </option>
                </select>
              </div>

              <!-- 语速 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text">语速: {{ t2aConfig.speed }}</span>
                </label>
                <input type="range" min="0.5" max="2.0" step="0.1" 
                       class="range range-primary" v-model.number="t2aConfig.speed">
                <div class="w-full flex justify-between text-xs px-2">
                  <span>0.5x</span>
                  <span>1.0x</span>
                  <span>2.0x</span>
                </div>
              </div>

              <!-- 音量 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text">音量: {{ t2aConfig.vol }}</span>
                </label>
                <input type="range" min="0.1" max="10" step="0.1" 
                       class="range range-primary" v-model.number="t2aConfig.vol">
                <div class="w-full flex justify-between text-xs px-2">
                  <span>0.1</span>
                  <span>5.0</span>
                  <span>10</span>
                </div>
              </div>

              <!-- 音高 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text">音高: {{ t2aConfig.pitch }}</span>
                </label>
                <input type="range" min="-12" max="12" step="1" 
                       class="range range-primary" v-model.number="t2aConfig.pitch">
                <div class="w-full flex justify-between text-xs px-2">
                  <span>-12</span>
                  <span>0</span>
                  <span>+12</span>
                </div>
              </div>

              <!-- 情感 -->
              <div class="form-control">
                <label class="label">
                  <span class="label-text">情感</span>
                </label>
                <select class="select select-bordered w-full" v-model="t2aConfig.emotion">
                  <option v-for="emotion in emotionOptions" :key="emotion.value" :value="emotion.value">
                    {{ emotion.label }}
                  </option>
                </select>
              </div>
            </div>
          </div>
        </div>

        <!-- 右侧文本输入区域 -->
        <div class="lg:col-span-2">
          <div class="card bg-base-200 shadow-sm h-full">
            <div class="card-body">
              <div class="flex justify-between items-center mb-4">
                <h2 class="card-title text-lg">文本输入</h2>
                <div class="text-sm" :class="{ 'text-error': isOverLimit }">
                  {{ charCount }} / {{ maxChars }}
                </div>
              </div>
              
              <textarea 
                class="textarea textarea-bordered w-full h-64 resize-none"
                :class="{ 'textarea-error': isOverLimit }"
                placeholder="请输入要合成的文本内容..."
                v-model="inputText"
                :maxlength="maxChars"
              ></textarea>
            </div>
          </div>
        </div>
      </div>

      <!-- 合成按钮 -->
      <div class="text-center mt-6">
        <button 
          class="btn btn-primary btn-lg"
          :class="{ 'loading': isLoading }"
          :disabled="isLoading || !inputText.trim() || isOverLimit"
          @click="startSynthesis"
        >
          {{ isLoading ? '合成中...' : '开始合成' }}
        </button>
      </div>

      <!-- 进度条 -->
      <div v-if="isLoading || progress > 0" class="mt-6">
        <div class="text-center mb-2">
          <span class="text-sm">合成进度: {{ progress }}%</span>
        </div>
        <progress class="progress progress-primary w-full" :value="progress" max="100"></progress>
      </div>

      <!-- 音频播放控件 -->
      <div v-if="audioUrl" class="mt-6">
        <div class="card bg-base-200 shadow-sm">
          <div class="card-body">
            <div class="flex justify-between items-center">
              <h3 class="text-lg font-semibold">音频播放</h3>
              <button class="btn btn-success" @click="downloadAudio">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
                </svg>
                下载
              </button>
            </div>
            <audio controls class="w-full mt-4" :src="audioUrl"></audio>
          </div>
        </div>
      </div>
    </div>

    <!-- 设置模态框 -->
    <div class="modal" :class="{ 'modal-open': showSettings }">
      <div class="modal-box">
        <h3 class="font-bold text-lg mb-4">API 配置</h3>
        
        <div class="form-control mb-4">
          <label class="label">
            <span class="label-text">API Key</span>
          </label>
          <input type="password" class="input input-bordered w-full" 
                 placeholder="请输入 MiniMax API Key" v-model="config.apiKey">
        </div>

        <div class="form-control mb-6">
          <label class="label">
            <span class="label-text">Group ID</span>
          </label>
          <input type="text" class="input input-bordered w-full" 
                 placeholder="请输入 Group ID" v-model="config.groupId">
        </div>

        <div class="modal-action">
          <button class="btn btn-ghost" @click="showSettings = false">取消</button>
          <button class="btn btn-primary" @click="saveConfig">保存</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.range {
  margin-bottom: 0.5rem;
}
</style>
