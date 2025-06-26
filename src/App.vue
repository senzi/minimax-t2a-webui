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
  speed: 1.0,
  vol: 1.0,
  pitch: 0,
  emotion: 'neutral'
})

// 音色混合配置（统一配置，单音色也使用此结构）
const timberWeights = reactive([
  { voiceId: 'male-qn-qingse', weight: 100 }
])

// 音色选择相关状态
const editingVoiceIndex = ref(-1) // 当前编辑的音色索引
const customVoiceMode = ref(false) // 是否为自定义音色模式
const customVoiceId = ref('') // 自定义音色ID输入
const customVoiceValidation = ref({ isValid: true, message: '' }) // 自定义音色验证结果

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


// 计算属性：是否为多音色模式
const isMultiVoiceMode = computed(() => {
  return timberWeights.length > 1
})

// 计算属性：权重总和
const totalWeight = computed(() => {
  return timberWeights.reduce((sum, item) => sum + (item.weight || 0), 0)
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
  loadTimberWeightsConfig()
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


// 验证自定义音色ID
function validateCustomVoiceId(voiceId) {
  if (!voiceId || voiceId.trim() === '') {
    return { isValid: false, message: '音色ID不能为空' }
  }
  
  const trimmedId = voiceId.trim()
  
  // 长度检查
  if (trimmedId.length < 1 || trimmedId.length > 50) {
    return { isValid: false, message: '音色ID长度应在1-50个字符之间' }
  }
  
  // 扩展的音色ID验证：支持字母、数字、下划线、连字符、空格和括号（全角半角）
  // 不能以数字开头
  const validPattern = /^[a-zA-Z_][a-zA-Z0-9_\-\s()（）]*$/
  if (!validPattern.test(trimmedId)) {
    return { isValid: false, message: '音色ID只能包含字母、数字、下划线、连字符、空格和括号，且不能以数字开头' }
  }
  
  return { isValid: true, message: '音色ID格式正确' }
}

// 实时验证自定义音色ID
function onCustomVoiceIdInput() {
  customVoiceValidation.value = validateCustomVoiceId(customVoiceId.value)
}

// 确认自定义音色ID
function confirmCustomVoice() {
  const validation = validateCustomVoiceId(customVoiceId.value)
  if (!validation.isValid) {
    customVoiceValidation.value = validation
    return
  }
  
  const trimmedId = customVoiceId.value.trim()
  selectVoice(trimmedId)
  
  // 重置自定义音色输入状态
  customVoiceMode.value = false
  customVoiceId.value = ''
  customVoiceValidation.value = { isValid: true, message: '' }
}

// 切换自定义音色模式
function toggleCustomVoiceMode() {
  customVoiceMode.value = !customVoiceMode.value
  if (customVoiceMode.value) {
    customVoiceId.value = ''
    customVoiceValidation.value = { isValid: true, message: '' }
  }
}

// 选择音色
function selectVoice(voiceId) {
  // 统一使用 timberWeights 配置
  if (editingVoiceIndex.value >= 0) {
    timberWeights[editingVoiceIndex.value].voiceId = voiceId
  } else {
    // 如果没有指定索引，默认修改第一个音色
    timberWeights[0].voiceId = voiceId
  }
  saveTimberWeightsConfig()
  showVoiceModal.value = false
  voiceSearchQuery.value = ''
  editingVoiceIndex.value = -1
}

// 打开音色选择模态框
function openVoiceModal(index = -1) {
  editingVoiceIndex.value = index
  showVoiceModal.value = true
  voiceSearchQuery.value = ''
}

// 添加音色到多音色配置
function addTimberWeight() {
  if (timberWeights.length >= 4) {
    showInlineAlert('最多只能添加 4 个音色', 'warning')
    return
  }
  
  timberWeights.push({
    voiceId: 'male-qn-qingse',
    weight: 50
  })
  saveTimberWeightsConfig()
}

// 删除音色配置项
function removeTimberWeight(index) {
  if (timberWeights.length <= 1) {
    showInlineAlert('至少需要保留一个音色', 'warning')
    return
  }
  
  timberWeights.splice(index, 1)
  saveTimberWeightsConfig()
}

// 获取音色信息
function getVoiceInfo(voiceId) {
  const systemVoice = systemVoices.find(voice => voice.voice_id === voiceId)
  if (systemVoice) {
    return systemVoice
  }
  
  // 自定义音色处理
  return {
    voice_id: voiceId,
    voice_name: '自定义音色',
    keywords: ['custom'],
    isCustom: true
  }
}

// 保存多音色配置
function saveTimberWeightsConfig() {
  localStorage.setItem('minimax-timber-weights', JSON.stringify(timberWeights))
}

// 加载多音色配置
function loadTimberWeightsConfig() {
  const saved = localStorage.getItem('minimax-timber-weights')
  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      if (Array.isArray(parsed) && parsed.length > 0) {
        timberWeights.splice(0, timberWeights.length, ...parsed)
      }
    } catch (e) {
      console.warn('加载多音色配置失败:', e)
    }
  }
}

// 校验多音色配置
function validateTimberWeights() {
  // 检查是否有空的音色ID或无效权重
  for (const item of timberWeights) {
    if (!item.voiceId || item.weight <= 0) {
      return false
    }
  }
  return true
}

// 构造请求体中的音色配置
function buildVoiceConfig() {
  if (!validateTimberWeights()) {
    throw new Error('请完整填写音色和权重')
  }

  // 归一化权重
  const total = timberWeights.reduce((sum, item) => sum + item.weight, 0)
  const normalized = timberWeights.map(item => ({
    voice_id: item.voiceId,
    weight: Math.round((item.weight / total) * 100)
  }))

  // 确保总和不超过100
  const sum = normalized.reduce((s, item) => s + item.weight, 0)
  if (sum > 100) {
    const maxIndex = normalized.findIndex(item => 
      item.weight === Math.max(...normalized.map(n => n.weight))
    )
    normalized[maxIndex].weight -= (sum - 100)
  }

  // 返回主音色ID和权重配置
  return {
    primaryVoiceId: timberWeights[0].voiceId,
    timber_weights: normalized
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
    // 构造音色配置
    const voiceConfig = buildVoiceConfig()
    
    // 构造请求体
    const requestBody = {
      model: t2aConfig.model,
      text: inputText.value,
      stream: true,
      output_format: 'hex',
      stream_options: {
        exclude_aggregated_audio: true
      },
      voice_setting: {
        voice_id: voiceConfig.primaryVoiceId, // 必填字段
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
      emotion: t2aConfig.emotion,
      timber_weights: voiceConfig.timber_weights
    }

    const response = await fetch(`https://api.minimaxi.com/v1/t2a_v2?GroupId=${config.groupId}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${config.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
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
                
                // 🔍 添加 hex 转换日志
                console.log('Hex string length:', hexString.length)
                console.log('First 20 chars of hex:', hexString.slice(0, 20))
                
                // 将 hex 字符串转换为 Uint8Array
                const audioData = new Uint8Array(hexString.length / 2)
                for (let i = 0; i < hexString.length; i += 2) {
                  audioData[i / 2] = parseInt(hexString.substr(i, 2), 16)
                }
                
                console.log('Converted audio chunk size:', audioData.length)
                audioChunks.push(audioData)
                
                // 更新接收块数和进度
                receivedChunks.value++
                progress.value = Math.floor((receivedChunks.value / expectedChunks.value) * 100)
                
                console.log(`🧱 添加音频块: ${audioChunks.length}, 大小: ${audioData.length}`)
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

    // --- 🔍 LOGGING BLOCK START ---
    console.log('🧱 当前音频块数:', audioChunks.length)
    console.log('📦 拼接前每块大小:', audioChunks.map(c => c.length))
    const totalLength = audioChunks.reduce((sum, chunk) => sum + chunk.length, 0)
    console.log('🧩 拼接后总长度:', totalLength)

    // 合并所有音频块
    const mergedAudio = new Uint8Array(totalLength)
    let offset = 0
    for (const chunk of audioChunks) {
      mergedAudio.set(chunk, offset)
      offset += chunk.length
    }

    audioBlob.value = new Blob([mergedAudio], { type: 'audio/mp3' })
    console.log('🎧 Blob size:', audioBlob.value.size)
    console.log('🎧 Blob type:', audioBlob.value.type)

    audioUrl.value = URL.createObjectURL(audioBlob.value)
    console.log('🔗 Audio URL:', audioUrl.value)
    // --- 🔍 LOGGING BLOCK END ---
    
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
      <div class="grid grid-cols-1 lg:grid-cols-5 gap-6">
        <!-- 左侧配置面板 -->
        <div class="lg:col-span-2">
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
              
              <!-- 音色及混合配置 -->
              <div class="form-control mb-4">
                <label class="label">
                  <span class="label-text text-base">音色及混合 <span class="text-xs text-base-content/60">(最多 4 个)</span></span>
                </label>
                
                <!-- 音色配置列表 -->
                <div class="space-y-2 mb-3">
                  <div 
                    v-for="(item, index) in timberWeights" 
                    :key="index"
                    class="flex items-center gap-2 p-2 bg-base-100 rounded-lg border border-base-300"
                  >
                    <!-- 音色选择按钮 -->
                    <button 
                      class="btn btn-outline btn-sm flex-1 justify-start text-left h-12 py-1"
                      @click="openVoiceModal(index)"
                    >
                      <div class="flex flex-col items-start w-full">
                        <div class="flex items-center gap-1">
                          <div class="font-medium text-xs">{{ getVoiceInfo(item.voiceId).voice_name }}</div>
                          <span 
                            v-if="getVoiceInfo(item.voiceId).isCustom" 
                            class="badge badge-xs badge-secondary"
                            title="自定义音色"
                          >
                            自定义
                          </span>
                        </div>
                        <div class="text-xs opacity-70">{{ item.voiceId }}</div>
                      </div>
                    </button>
                    
                    <!-- 权重输入 -->
                    <div class="flex items-center gap-1">
                      <input 
                        type="number" 
                        class="input input-bordered input-sm w-16 text-center text-xs"
                        min="1"
                        step="1"
                        v-model.number="item.weight"
                        @input="saveTimberWeightsConfig"
                      >
                      <span class="text-xs opacity-70">%</span>
                    </div>
                    
                    <!-- 删除按钮 -->
                    <button 
                      v-if="timberWeights.length > 1"
                      class="btn btn-ghost btn-sm btn-circle text-error hover:bg-error hover:text-error-content"
                      @click="removeTimberWeight(index)"
                      title="删除此音色"
                    >
                      <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                      </svg>
                    </button>
                  </div>
                </div>
                
                <!-- 添加音色按钮 -->
                <button 
                  v-if="timberWeights.length < 4"
                  class="btn btn-outline btn-sm w-full"
                  @click="addTimberWeight"
                >
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6" />
                  </svg>
                  添加音色
                </button>
                
                <!-- 权重总和提示 -->
                <div v-if="isMultiVoiceMode" class="text-xs text-base-content/60 mt-2">
                  权重总和: {{ totalWeight }}% (提交时会自动归一化)
                </div>
              </div>

              <!-- 语速 -->
              <div class="form-control mb-3">
                <div class="flex items-center gap-3">
                  <div class="flex-shrink-0 w-20">
                    <span class="text-sm font-medium">语速</span>
                    <div class="text-xs text-base-content/70">{{ t2aConfig.speed }}</div>
                  </div>
                  <div class="flex-1">
                    <input type="range" min="0.5" max="2.0" step="0.1" 
                           class="range range-primary range-sm" v-model.number="t2aConfig.speed">
                  </div>
                </div>
              </div>

              <!-- 音量 -->
              <div class="form-control mb-3">
                <div class="flex items-center gap-3">
                  <div class="flex-shrink-0 w-20">
                    <span class="text-sm font-medium">音量</span>
                    <div class="text-xs text-base-content/70">{{ t2aConfig.vol }}</div>
                  </div>
                  <div class="flex-1">
                    <input type="range" min="0.1" max="10" step="0.1" 
                           class="range range-primary range-sm" v-model.number="t2aConfig.vol">
                  </div>
                </div>
              </div>

              <!-- 音高 -->
              <div class="form-control mb-3">
                <div class="flex items-center gap-3">
                  <div class="flex-shrink-0 w-20">
                    <span class="text-sm font-medium">音高</span>
                    <div class="text-xs text-base-content/70">{{ t2aConfig.pitch }}</div>
                  </div>
                  <div class="flex-1">
                    <input type="range" min="-12" max="12" step="1" 
                           class="range range-primary range-sm" v-model.number="t2aConfig.pitch">
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
        <div class="lg:col-span-3">
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
        
        <!-- 模式切换按钮 -->
        <div class="tabs tabs-boxed mb-4">
          <button 
            class="tab text-sm"
            :class="{ 'tab-active': !customVoiceMode }"
            @click="toggleCustomVoiceMode"
          >
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10" />
            </svg>
            选择预设音色
          </button>
          <button 
            class="tab text-sm"
            :class="{ 'tab-active': customVoiceMode }"
            @click="toggleCustomVoiceMode"
          >
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
            </svg>
            输入自定义音色ID
          </button>
        </div>

        <!-- 自定义音色输入区域 -->
        <div v-if="customVoiceMode" class="mb-6">
          <div class="card bg-base-200 border border-base-300">
            <div class="card-body p-4">
              <h4 class="font-semibold text-base mb-3">自定义音色ID</h4>
              
              <!-- 输入框 -->
              <div class="form-control mb-3">
                <input 
                  type="text" 
                  class="input input-bordered w-full text-base"
                  :class="{ 'input-error': !customVoiceValidation.isValid && customVoiceId.trim() !== '' }"
                  placeholder="请输入自定义音色ID，如：my_custom_voice"
                  v-model="customVoiceId"
                  @input="onCustomVoiceIdInput"
                  @keyup.enter="confirmCustomVoice"
                >
              </div>
              
              <!-- 验证提示 -->
              <div class="mb-3">
                <div 
                  v-if="customVoiceId.trim() !== ''"
                  class="text-sm"
                  :class="{
                    'text-success': customVoiceValidation.isValid,
                    'text-error': !customVoiceValidation.isValid
                  }"
                >
                  <svg 
                    xmlns="http://www.w3.org/2000/svg" 
                    class="h-4 w-4 inline mr-1" 
                    fill="none" 
                    viewBox="0 0 24 24" 
                    stroke="currentColor"
                  >
                    <path 
                      v-if="customVoiceValidation.isValid"
                      stroke-linecap="round" 
                      stroke-linejoin="round" 
                      stroke-width="2" 
                      d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" 
                    />
                    <path 
                      v-else
                      stroke-linecap="round" 
                      stroke-linejoin="round" 
                      stroke-width="2" 
                      d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L3.732 16c-.77.833.192 2.5 1.732 2.5z" 
                    />
                  </svg>
                  {{ customVoiceValidation.message }}
                </div>
              </div>
              
              <!-- 规则说明 -->
              <div class="alert alert-info text-sm mb-3">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" class="stroke-current shrink-0 w-4 h-4">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
                </svg>
                <div>
                  <div class="font-medium">音色ID规则：</div>
                  <ul class="text-xs mt-1 space-y-1">
                    <li>• 只能包含字母、数字、下划线(_)、连字符(-)、空格和括号</li>
                    <li>• 支持全角和半角括号：() （）</li>
                    <li>• 不能以数字开头</li>
                    <li>• 长度在1-50个字符之间</li>
                  </ul>
                </div>
              </div>
              
              <!-- 确认按钮 -->
              <button 
                class="btn btn-primary btn-sm"
                :disabled="!customVoiceValidation.isValid || customVoiceId.trim() === ''"
                @click="confirmCustomVoice"
              >
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                </svg>
                确认使用此音色ID
              </button>
            </div>
          </div>
        </div>

        <!-- 预设音色选择区域 -->
        <div v-else>
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
                :class="{ 'btn-primary': voice.voice_id === (editingVoiceIndex >= 0 ? timberWeights[editingVoiceIndex]?.voiceId : timberWeights[0]?.voiceId) }"
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

/* 情感按钮网格布局 */
.emotion-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.5rem;
}

/* 自动调整高度的文本框 */
.auto-resize-textarea {
  min-height: 120px;
  resize: none;
  transition: height 0.2s ease;
}

.auto-resize-textarea.scrollable {
  overflow-y: auto;
}

/* 导航栏样式 */
.navbar-wrapper {
  position: sticky;
  top: 0;
  z-index: 40;
  background: hsl(var(--b1));
  border-bottom: 1px solid hsl(var(--b3));
}

/* 多音色配置项的悬停效果 */
.timber-weight-item {
  transition: all 0.2s ease;
}

.timber-weight-item:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

/* 权重输入框样式优化 */
.weight-input {
  transition: border-color 0.2s ease;
}

.weight-input:focus {
  border-color: hsl(var(--p));
  box-shadow: 0 0 0 2px hsl(var(--p) / 0.2);
}

/* 删除按钮的悬停动画 */
.delete-btn {
  transition: all 0.2s ease;
  opacity: 0.7;
}

.delete-btn:hover {
  opacity: 1;
  transform: scale(1.1);
}

/* 添加音色按钮样式 */
.add-voice-btn {
  transition: all 0.2s ease;
  border-style: dashed;
}

.add-voice-btn:hover {
  border-style: solid;
  transform: translateY(-1px);
}

/* 权重总和提示样式 */
.weight-summary {
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', monospace;
}
</style>
