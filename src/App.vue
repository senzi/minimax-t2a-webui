<script setup>
import { ref, reactive, onMounted, computed } from 'vue'
import systemVoices from './assets/voices/system_voice.json'

// 响应式数据
const showSettings = ref(false)
const showVoiceModal = ref(false)
const isLoading = ref(false)
const audioUrl = ref('')
const audioBlob = ref(null)
const progress = ref(0)
const voiceSearchQuery = ref('')

// 新增：字符使用量和费用相关
const usageChars = ref(0)
const receivedChunks = ref(0)
const expectedChunks = ref(20) // 默认预估值

// 内联提示相关
const alertMessage = ref('')
const alertType = ref('error') // 'error', 'warning', 'success', 'info'
const showAlert = ref(false)

// 配置数据
const config = reactive({
  apiKey: '',
  groupId: ''
})

// T2A 参数配置
const t2aConfig = reactive({
  model: 'speech-02-hd',
  voiceId: 'male-qn-qingse',
  speed: 1.0,
  vol: 1.0,
  pitch: 0,
  emotion: 'neutral'
})

// 文本输入
const inputText = ref('')
const maxChars = 5000
const textareaRef = ref(null)
const isTextareaScrollable = ref(false)

// 模型选项
const modelOptions = [
  { id: 'speech-02-hd', name: 'Speech-02-HD', description: '持续更新的HD模型，拥有更出色的韵律、稳定性和复刻相似度，音质表现突出' },
  { id: 'speech-02-turbo', name: 'Speech-02-Turbo', description: '持续更新的Turbo模型，拥有更出色的韵律和稳定性，小语种能力加强，性能表现出色' },
  { id: 'speech-01-hd', name: 'Speech-01-HD', description: '稳定版本的HD模型，拥有超高的复刻相似度，音质表现突出' },
  { id: 'speech-01-turbo', name: 'Speech-01-Turbo', description: '稳定版本的Turbo模型，在出色的生成效果基础上有更快的生成速度' }
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

// 计算属性：当前选中的音色信息
const currentVoice = computed(() => {
  return systemVoices.find(voice => voice.voice_id === t2aConfig.voiceId) || {
    voice_id: 'male-qn-qingse',
    voice_name: '青涩青年音色',
    keywords: ['male', 'qingse', 'qn', '青涩青年音色']
  }
})

// 计算属性：过滤后的音色列表
const filteredVoices = computed(() => {
  if (!voiceSearchQuery.value.trim()) {
    return systemVoices
  }
  
  const query = voiceSearchQuery.value.toLowerCase().trim()
  return systemVoices.filter(voice => {
    // 搜索 voice_name
    if (voice.voice_name.toLowerCase().includes(query)) {
      return true
    }
    
    // 搜索 voice_id
    if (voice.voice_id.toLowerCase().includes(query)) {
      return true
    }
    
    // 搜索 keywords
    if (voice.keywords.some(keyword => keyword.toLowerCase().includes(query))) {
      return true
    }
    
    return false
  })
})

// 计算属性：费用估算
const estimatedCost = computed(() => {
  return ((usageChars.value / 10000) * 3.5).toFixed(2)
})

// 计算属性：进度描述
const progressLabel = computed(() => {
  return `正在处理第 ${receivedChunks.value} 块 / 预计 ${expectedChunks.value} 块`
})

// 页面初始化
onMounted(() => {
  loadConfig()
  loadVoiceConfig()
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

// 加载音色配置
function loadVoiceConfig() {
  const savedVoice = localStorage.getItem('minimax-voice')
  if (savedVoice) {
    const voiceExists = systemVoices.find(voice => voice.voice_id === savedVoice)
    if (voiceExists) {
      t2aConfig.voiceId = savedVoice
    }
  }
}

// 保存音色配置
function saveVoiceConfig() {
  localStorage.setItem('minimax-voice', t2aConfig.voiceId)
}

// 选择音色
function selectVoice(voiceId) {
  t2aConfig.voiceId = voiceId
  saveVoiceConfig()
  showVoiceModal.value = false
  voiceSearchQuery.value = ''
}

// 打开音色选择模态框
function openVoiceModal() {
  showVoiceModal.value = true
  voiceSearchQuery.value = ''
}

// 保存配置
function saveConfig() {
  localStorage.setItem('minimax-config', JSON.stringify({
    apiKey: config.apiKey,
    groupId: config.groupId
  }))
  showSettings.value = false
  // 显示保存成功提示
  showInlineAlert('配置保存成功！', 'success')
}

// 开始合成
async function startSynthesis() {
  if (!inputText.value.trim()) {
    showInlineAlert('请输入要合成的文本', 'warning')
    return
  }
  
  if (!config.apiKey || !config.groupId) {
    showInlineAlert('请先配置 API Key 和 Group ID', 'warning')
    showSettings.value = true
    return
  }

  isLoading.value = true
  progress.value = 0
  audioUrl.value = ''
  audioBlob.value = null
  
  // 合成开始前，预估 usageChars 和 expectedChunks
  const estimatedChars = estimateUsageCharacters(inputText.value)
  usageChars.value = estimatedChars

  const estimatedPerChunk = 16 // 平均每 16 字符一个数据块（由实际观测得出）
  expectedChunks.value = Math.ceil(estimatedChars / estimatedPerChunk)
  
  // 重置接收块数计数器
  receivedChunks.value = 0

  console.log(`预测字符数: ${usageChars.value}，预计数据块: ${expectedChunks.value}`)

  try {
    const response = await fetch(`https://api.minimaxi.com/v1/t2a_v2?GroupId=${config.groupId}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${config.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: t2aConfig.model,
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
          channel: 2
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
                
                // 将 hex 字符串转换为 Uint8Array
                const audioData = new Uint8Array(hexString.length / 2)
                for (let i = 0; i < hexString.length; i += 2) {
                  audioData[i / 2] = parseInt(hexString.substr(i, 2), 16)
                }
                audioChunks.push(audioData)
                
                // 更新接收块数和进度
                receivedChunks.value++
                progress.value = Math.floor((receivedChunks.value / expectedChunks.value) * 100)
                
                console.log(`添加音频块: ${audioChunks.length}, 大小: ${audioData.length}`)
              }

              // 检查是否完成，提取使用字符数
              if (data.data && data.data.status === 2) {
                progress.value = 100
                if (data.extra_info && typeof data.extra_info.usage_characters === 'number') {
                  usageChars.value = data.extra_info.usage_characters

                  // 基于经验字符密度修正预计块数
                  const estimatedPerChunk = 16
                  expectedChunks.value = Math.ceil(usageChars.value / estimatedPerChunk)
                  console.log(`使用字符：${usageChars.value}，推算预计块数：${expectedChunks.value}`)
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
            
            const audioData = new Uint8Array(hexString.length / 2)
            for (let i = 0; i < hexString.length; i += 2) {
              audioData[i / 2] = parseInt(hexString.substr(i, 2), 16)
            }
            audioChunks.push(audioData)
            receivedChunks.value++
            console.log(`缓冲区添加音频块: ${audioChunks.length}, 大小: ${audioData.length}`)
          }
          
          // 检查缓冲区数据是否包含完成状态和使用字符数
          if (data.data && data.data.status === 2) {
            if (data.extra_info && typeof data.extra_info.usage_characters === 'number') {
              usageChars.value = data.extra_info.usage_characters

              // 基于经验字符密度修正预计块数
              const estimatedPerChunk = 16
              expectedChunks.value = Math.ceil(usageChars.value / estimatedPerChunk)
              console.log(`使用字符：${usageChars.value}，推算预计块数：${expectedChunks.value}`)
            }
          }
        }
      } catch (e) {
        console.warn('解析缓冲区数据失败:', e)
      }
    }

    console.log(`共接收到 ${receivedChunks.value} 块数据，预计总块数为 ${expectedChunks.value}`)

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
    showInlineAlert('合成失败: ' + error.message, 'error')
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

// 调整文本框高度
function adjustTextareaHeight() {
  if (!textareaRef.value) return
  
  const textarea = textareaRef.value
  const maxHeight = 640 // 40rem = 640px
  
  // 重置高度以获取正确的 scrollHeight
  textarea.style.height = 'auto'
  
  // 计算需要的高度
  const scrollHeight = textarea.scrollHeight
  
  if (scrollHeight <= maxHeight) {
    // 未达到最大高度，自动扩高，隐藏滚动条
    textarea.style.height = scrollHeight + 'px'
    isTextareaScrollable.value = false
  } else {
    // 达到最大高度，显示滚动条
    textarea.style.height = maxHeight + 'px'
    isTextareaScrollable.value = true
  }
}

// 字符计数
const charCount = computed(() => inputText.value.length)
const isOverLimit = computed(() => charCount.value > maxChars)

// 显示内联提示
function showInlineAlert(message, type = 'error') {
  alertMessage.value = message
  alertType.value = type
  showAlert.value = true
  
  // 自动隐藏提示（错误和警告类型保持更长时间）
  const duration = type === 'error' || type === 'warning' ? 5000 : 3000
  setTimeout(() => {
    showAlert.value = false
  }, duration)
}

// 隐藏内联提示
function hideInlineAlert() {
  showAlert.value = false
}

// 估算字符使用量（与官方计费规则一致：1汉字=2字符，其余=1字符）
function estimateUsageCharacters(text) {
  let count = 0
  for (const ch of text) {
    const code = ch.charCodeAt(0)
    if (code >= 0x4e00 && code <= 0x9fa5) {
      count += 2 // 汉字
    } else {
      count += 1 // 其他字符
    }
  }
  return count
}
</script>

<template>
  <div class="min-h-screen bg-base-100">
    <!-- 顶部标题栏 -->
    <div class="navbar-wrapper">
      <div class="navbar">
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
    </div>

    <!-- 主要内容区域 -->
    <div class="container mx-auto p-4">
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- 左侧配置面板 -->
        <div class="lg:col-span-1">
          <div class="card bg-base-200 shadow-sm">
            <div class="card-body">
              <h2 class="card-title text-lg mb-4">语音配置</h2>
              
              <!-- 模型选择 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text text-base">模型</span>
                </label>
                <div class="grid grid-cols-2 gap-2 mb-2">
                  <button 
                    v-for="model in modelOptions" 
                    :key="model.id"
                    class="btn btn-sm text-sm"
                    :class="t2aConfig.model === model.id ? 'btn-primary' : 'btn-outline'"
                    @click="t2aConfig.model = model.id"
                  >
                    {{ model.name }}
                  </button>
                </div>
                <div class="text-sm text-base-content/70 leading-relaxed">
                  {{ modelOptions.find(m => m.id === t2aConfig.model)?.description }}
                </div>
              </div>
              
              <!-- 音色选择 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text text-base">音色</span>
                </label>
                <button 
                  class="btn btn-outline w-full justify-start text-left h-16 py-2"
                  @click="openVoiceModal"
                >
                  <div class="flex flex-col items-start w-full">
                    <div class="font-medium text-sm">{{ currentVoice.voice_name }}</div>
                    <div class="text-xs opacity-70">{{ currentVoice.voice_id }}</div>
                  </div>
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 ml-auto flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                  </svg>
                </button>
              </div>

              <!-- 语速 -->
              <div class="form-control mb-6">
                <label class="label py-3">
                  <span class="label-text text-base">语速: {{ t2aConfig.speed }}</span>
                </label>
                <div class="px-3">
                  <input type="range" min="0.5" max="2.0" step="0.1" 
                         class="range range-primary my-2" v-model.number="t2aConfig.speed">
                  <div class="w-full flex justify-between text-sm px-2 mt-2">
                    <span>0.5x</span>
                    <span>1.0x</span>
                    <span>2.0x</span>
                  </div>
                </div>
              </div>

              <!-- 音量 -->
              <div class="form-control mb-6">
                <label class="label py-3">
                  <span class="label-text text-base">音量: {{ t2aConfig.vol }}</span>
                </label>
                <div class="px-3">
                  <input type="range" min="0.1" max="10" step="0.1" 
                         class="range range-primary my-2" v-model.number="t2aConfig.vol">
                  <div class="w-full flex justify-between text-sm px-2 mt-2">
                    <span>0.1</span>
                    <span>5.0</span>
                    <span>10</span>
                  </div>
                </div>
              </div>

              <!-- 音高 -->
              <div class="form-control mb-6">
                <label class="label py-3">
                  <span class="label-text text-base">音高: {{ t2aConfig.pitch }}</span>
                </label>
                <div class="px-3">
                  <input type="range" min="-12" max="12" step="1" 
                         class="range range-primary my-2" v-model.number="t2aConfig.pitch">
                  <div class="w-full flex justify-between text-sm px-2 mt-2">
                    <span>-12</span>
                    <span>0</span>
                    <span>+12</span>
                  </div>
                </div>
              </div>

              <!-- 情感 -->
              <div class="form-control">
                <label class="label">
                  <span class="label-text text-base">情感</span>
                </label>
                <div class="emotion-grid">
                  <button 
                    v-for="emotion in emotionOptions" 
                    :key="emotion.value"
                    class="btn btn-sm text-sm"
                    :class="t2aConfig.emotion === emotion.value ? 'btn-primary' : 'btn-outline'"
                    @click="t2aConfig.emotion = emotion.value"
                  >
                    {{ emotion.label }}
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 右侧文本输入区域 -->
        <div class="lg:col-span-2">
          <div class="card bg-base-200 shadow-sm">
            <div class="card-body">
              <div class="flex justify-between items-center mb-4">
                <h2 class="card-title text-xl">文本输入</h2>
                <div class="text-base" :class="{ 'text-error': isOverLimit }">
                  {{ charCount }} / {{ maxChars }}
                </div>
              </div>
              
              <textarea 
                ref="textareaRef"
                class="textarea textarea-bordered w-full auto-resize-textarea text-lg leading-relaxed"
                :class="{ 'textarea-error': isOverLimit, 'scrollable': isTextareaScrollable }"
                placeholder="请输入要合成的文本内容..."
                v-model="inputText"
                :maxlength="maxChars"
                @input="adjustTextareaHeight"
              ></textarea>

              <!-- 合成按钮 -->
              <div class="mt-6">
                <button 
                  class="btn btn-primary btn-lg text-lg w-full"
                  :class="{ 'loading': isLoading }"
                  :disabled="isLoading || !inputText.trim() || isOverLimit"
                  @click="startSynthesis"
                >
                  {{ isLoading ? '合成中...' : '开始合成' }}
                </button>
              </div>

              <!-- 进度条 -->
              <div v-if="isLoading || progress > 0" class="mt-4">
                <div v-if="isLoading || progress > 0" class="mb-2 text-sm text-base-content/60">
                  {{ progressLabel }}
                </div>
                <div class="mb-2">
                  <span class="text-base">合成进度: {{ progress }}%</span>
                </div>
                <progress class="progress progress-primary w-full h-3" :value="progress" max="100"></progress>
              </div>

              <!-- 音频播放控件 -->
              <div v-if="audioUrl" class="mt-6">
                <div class="card bg-base-300 shadow-sm">
                  <div class="card-body py-4">
                    <div class="flex justify-between items-center mb-4">
                      <h3 class="text-lg font-semibold">音频播放</h3>
                      <button class="btn btn-success text-base" @click="downloadAudio">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
                        </svg>
                        下载
                      </button>
                    </div>
                    <div v-if="usageChars > 0" class="text-sm text-base-content/70 mb-2">
                      实际使用字符：{{ usageChars }}，预估费用：¥{{ estimatedCost }}
                    </div>
                    <audio controls class="w-full h-12" :src="audioUrl"></audio>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

    </div>

    <!-- 内联提示组件 -->
    <div v-if="showAlert" class="fixed top-4 left-1/2 transform -translate-x-1/2 z-50 w-full max-w-md px-4">
      <div class="alert shadow-lg" 
           :class="{
             'alert-error': alertType === 'error',
             'alert-warning': alertType === 'warning', 
             'alert-success': alertType === 'success',
             'alert-info': alertType === 'info'
           }">
        <div class="flex items-center justify-between w-full">
          <div class="flex items-center">
            <!-- 错误图标 -->
            <svg v-if="alertType === 'error'" xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6 mr-2" fill="none" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <!-- 警告图标 -->
            <svg v-else-if="alertType === 'warning'" xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6 mr-2" fill="none" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L3.732 16c-.77.833.192 2.5 1.732 2.5z" />
            </svg>
            <!-- 成功图标 -->
            <svg v-else-if="alertType === 'success'" xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6 mr-2" fill="none" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <!-- 信息图标 -->
            <svg v-else xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6 mr-2" fill="none" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <span class="text-sm font-medium">{{ alertMessage }}</span>
          </div>
          <button class="btn btn-ghost btn-sm btn-circle ml-2" @click="hideInlineAlert">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
            </svg>
          </button>
        </div>
      </div>
    </div>

    <!-- 设置模态框 -->
    <div class="modal" :class="{ 'modal-open': showSettings }">
      <div class="modal-box max-w-2xl">
        <h3 class="font-bold text-xl mb-4">API 配置</h3>
        
        <!-- 引导说明 -->
        <div class="alert alert-info mb-6">
          <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" class="stroke-current shrink-0 w-6 h-6">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
          </svg>
          <div>
            <h4 class="font-bold">配置说明</h4>
            <div class="text-sm mt-1">请按照以下步骤获取所需的配置信息</div>
          </div>
        </div>

        <!-- Group ID 配置 -->
        <div class="form-control mb-4">
          <label class="label">
            <span class="label-text text-base font-semibold">Group ID</span>
          </label>
          <input type="text" class="input input-bordered w-full text-base" 
                 placeholder="请输入 Group ID" v-model="config.groupId">
          <div class="label">
            <span class="label-text-alt text-sm">
              <div class="flex items-center gap-2 mt-2">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-info" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                </svg>
                <span>获取方式：登录 MiniMax 平台，在</span>
                <a href="https://platform.minimaxi.com/user-center/basic-information" 
                   target="_blank" 
                   class="link link-primary font-medium">
                  基本信息页面
                </a>
                <span>查看</span>
              </div>
            </span>
          </div>
        </div>

        <!-- API Key 配置 -->
        <div class="form-control mb-6">
          <label class="label">
            <span class="label-text text-base font-semibold">API Key</span>
          </label>
          <input type="password" class="input input-bordered w-full text-base" 
                 placeholder="请输入 MiniMax API Key" v-model="config.apiKey">
          <div class="label">
            <span class="label-text-alt text-sm">
              <div class="flex items-start gap-2 mt-2">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-warning mt-0.5 flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L3.732 16c-.77.833.192 2.5 1.732 2.5z" />
                </svg>
                <div>
                  <div class="mb-1">
                    <span>获取方式：前往</span>
                    <a href="https://platform.minimaxi.com/user-center/basic-information/interface-key" 
                       target="_blank" 
                       class="link link-primary font-medium">
                      接口密钥页面
                    </a>
                    <span>申请</span>
                  </div>
                  <div class="text-warning font-medium">
                    ⚠️ 重要：申请后请立即复制保存，页面不会重复显示！
                  </div>
                </div>
              </div>
            </span>
          </div>
        </div>

        <!-- 配置步骤说明 -->
        <div class="collapse collapse-arrow bg-base-200 mb-4">
          <input type="checkbox" /> 
          <div class="collapse-title text-base font-medium">
            📋 详细配置步骤
          </div>
          <div class="collapse-content"> 
            <div class="space-y-3 text-sm">
              <div class="steps steps-vertical">
                <div class="step step-primary">
                  <div class="text-left">
                    <div class="font-semibold">获取 Group ID</div>
                    <div class="text-xs opacity-70 mt-1">
                      访问 <a href="https://platform.minimaxi.com/user-center/basic-information" target="_blank" class="link">基本信息页面</a>，
                      在页面中找到您的 Group ID
                    </div>
                  </div>
                </div>
                <div class="step step-primary">
                  <div class="text-left">
                    <div class="font-semibold">申请 API Key</div>
                    <div class="text-xs opacity-70 mt-1">
                      访问 <a href="https://platform.minimaxi.com/user-center/basic-information/interface-key" target="_blank" class="link">接口密钥页面</a>，
                      点击申请新的 API Key
                    </div>
                  </div>
                </div>
                <div class="step step-primary">
                  <div class="text-left">
                    <div class="font-semibold">保存配置</div>
                    <div class="text-xs opacity-70 mt-1">
                      将获取到的 Group ID 和 API Key 填入上方表单，点击保存
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="modal-action">
          <button class="btn btn-ghost text-base" @click="showSettings = false">取消</button>
          <button class="btn btn-primary text-base" @click="saveConfig">保存配置</button>
        </div>
      </div>
    </div>

    <!-- 音色选择模态框 -->
    <div class="modal" :class="{ 'modal-open': showVoiceModal }">
      <div class="modal-box max-w-4xl">
        <h3 class="font-bold text-xl mb-4">选择音色</h3>
        
        <!-- 搜索框 -->
        <div class="form-control mb-4">
          <input 
            type="text" 
            class="input input-bordered w-full text-base" 
            placeholder="搜索音色名称、ID或关键词..."
            v-model="voiceSearchQuery"
          >
        </div>

        <!-- 音色网格 -->
        <div class="max-h-96 overflow-y-auto">
          <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
            <button
              v-for="voice in filteredVoices"
              :key="voice.voice_id"
              class="btn btn-outline text-left h-auto p-3"
              :class="{ 'btn-primary': voice.voice_id === t2aConfig.voiceId }"
              @click="selectVoice(voice.voice_id)"
            >
              <div class="flex flex-col items-start w-full">
                <div class="font-medium text-sm">{{ voice.voice_name }}</div>
                <div class="text-xs opacity-70 mt-1">{{ voice.voice_id }}</div>
                <div class="text-xs opacity-50 mt-1 flex flex-wrap gap-1">
                  <span 
                    v-for="keyword in voice.keywords.slice(0, 3)" 
                    :key="keyword"
                    class="badge badge-xs"
                  >
                    {{ keyword }}
                  </span>
                  <span v-if="voice.keywords.length > 3" class="text-xs">...</span>
                </div>
              </div>
            </button>
          </div>
          
          <!-- 无搜索结果提示 -->
          <div v-if="filteredVoices.length === 0" class="text-center py-8 text-base-content/50">
            <div class="text-lg mb-2">😔</div>
            <div>未找到匹配的音色</div>
            <div class="text-sm mt-1">请尝试其他关键词</div>
          </div>
        </div>

        <div class="modal-action">
          <button class="btn btn-ghost text-base" @click="showVoiceModal = false">关闭</button>
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
