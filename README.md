# MiniMax T2A Web UI

一个基于 Vue 3 + Vite + Tailwind CSS + daisyUI 的 MiniMax 文本转语音 Web 界面。

## 🚀 功能特性

- **文本转语音合成**：支持 MiniMax T2A API 的流式音频合成
- **实时配置**：音色、语速、音量、音高、情感等参数可调
- **流式播放**：音频数据边收边播，支持实时播放
- **音频下载**：自动生成文件名，支持 MP3 格式下载
- **配置管理**：API Key 和 Group ID 本地存储
- **响应式设计**：适配桌面和移动端

## 🛠️ 技术栈

- **前端框架**：Vue 3 (Composition API + `<script setup>`)
- **构建工具**：Vite
- **UI 框架**：Tailwind CSS + daisyUI
- **音频处理**：原生 Web Audio API + Fetch Stream

## 📦 安装与运行

### 环境要求
- Node.js 16+
- npm 或 yarn

### 安装依赖
```bash
npm install
```

### 开发模式
```bash
npm run dev
```

### 构建生产版本
```bash
npm run build
```

### 预览生产版本
```bash
npm run preview
```

## 🔧 配置说明

### API 配置
1. 点击右上角设置按钮
2. 输入 MiniMax API Key
3. 输入 Group ID
4. 点击保存（配置会保存到浏览器本地存储）

### 语音参数
- **音色**：支持多种预设音色（青年男声、少女、青叔、甜美女声等）
- **语速**：0.5x - 2.0x 可调
- **音量**：0.1 - 10 可调
- **音高**：-12 到 +12 半音可调
- **情感**：中性、开心、悲伤、愤怒、恐惧、厌恶、惊讶

## 📝 使用说明

1. **配置 API**：首次使用需要配置 MiniMax API Key 和 Group ID
2. **调整参数**：在左侧面板调整语音合成参数
3. **输入文本**：在右侧文本框输入要合成的内容（最多 5000 字符）
4. **开始合成**：点击"开始合成"按钮
5. **播放音频**：合成完成后可直接播放
6. **下载文件**：点击下载按钮保存 MP3 文件

## 🎵 音频格式

- **格式**：MP3
- **采样率**：32000 Hz
- **比特率**：128 kbps
- **声道**：单声道

## 📁 文件命名规则

下载的音频文件按以下规则命名：
```
[文本前5个有效字符]-[HH_MM].mp3
```
例如：`真正的危-14_30.mp3`

## 🔗 API 接口

使用 MiniMax T2A v2 接口：
- **接口地址**：`https://api.minimaxi.com/v1/t2a_v2`
- **认证方式**：Bearer Token
- **数据格式**：流式 event-stream，hex 编码音频数据

## 🌐 浏览器兼容性

- Chrome 88+
- Firefox 90+
- Safari 14+
- Edge 88+

## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📞 支持

如有问题，请查看 [MiniMax API 文档](https://api.minimaxi.com/document/guides/text-to-speech) 或提交 Issue。
