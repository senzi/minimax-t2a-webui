# MiniMax T2A Web UI

一个基于 Vue 3 + Vite + Tailwind CSS + daisyUI 的 MiniMax 文本转语音 Web 界面。

## 🌐 在线体验

**Live Demo**: [https://voice.closeai.moe/](https://voice.closeai.moe/)

## 🚀 功能特性

- **多模型支持**：支持 Speech-02-HD、Speech-02-Turbo、Speech-01-HD、Speech-01-Turbo 四种模型
- **音色混合**：支持最多 4 个音色混合，可自定义权重比例
- **自定义音色**：支持输入自定义音色 ID，扩展音色选择范围
- **文本转语音合成**：支持 MiniMax T2A API 的流式音频合成
- **实时配置**：音色、语速、音量、音高、情感等参数可调
- **流式播放**：音频数据边收边播，支持实时播放和进度显示
- **音频下载**：自动生成文件名，支持 MP3 格式下载
- **费用估算**：实时显示字符使用量和预估费用
- **配置管理**：API Key 和 Group ID 本地存储
- **更新日志**：内置更新日志查看功能
- **响应式设计**：适配桌面和移动端

## 🛠️ 技术栈

- **前端框架**：Vue 3 (Composition API + `<script setup>`)
- **构建工具**：Vite
- **UI 框架**：Tailwind CSS + daisyUI
- **音频处理**：原生 Web Audio API + Fetch Stream
- **Markdown 渲染**：marked

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
2. 获取 Group ID：访问 [MiniMax 基本信息页面](https://platform.minimaxi.com/user-center/basic-information)
3. 申请 API Key：访问 [MiniMax 接口密钥页面](https://platform.minimaxi.com/user-center/basic-information/interface-key)
4. 填入配置信息并保存（配置会保存到浏览器本地存储）

### 模型选择
- **Speech-02-HD**：持续更新的HD模型，拥有更出色的韵律、稳定性和复刻相似度，音质表现突出
- **Speech-02-Turbo**：持续更新的Turbo模型，拥有更出色的韵律和稳定性，小语种能力加强，性能表现出色
- **Speech-01-HD**：稳定版本的HD模型，拥有超高的复刻相似度，音质表现突出
- **Speech-01-Turbo**：稳定版本的Turbo模型，在出色的生成效果基础上有更快的生成速度

### 音色配置
- **预设音色**：支持多种预设音色选择，包含音色名称、ID 和关键词搜索
- **自定义音色**：支持输入自定义音色 ID，规则如下：
  - 只能包含字母、数字、下划线(_)、连字符(-)、空格和括号
  - 支持全角和半角括号：() （）
  - 不能以数字开头
  - 长度在1-50个字符之间
- **音色混合**：最多可选择 4 个音色进行混合，支持权重配置（自动归一化）

### 语音参数
- **语速**：0.5x - 2.0x 可调
- **音量**：0.1 - 10 可调
- **音高**：-12 到 +12 半音可调
- **情感**：中性、开心、悲伤、愤怒、恐惧、厌恶、惊讶

## 📝 使用说明

1. **配置 API**：首次使用需要配置 MiniMax API Key 和 Group ID
2. **选择模型**：根据需求选择合适的语音合成模型
3. **配置音色**：选择单个音色或配置多音色混合
4. **调整参数**：在左侧面板调整语音合成参数
5. **输入文本**：在右侧文本框输入要合成的内容（最多 5000 字符）
6. **开始合成**：点击"开始合成"按钮，可实时查看进度
7. **播放音频**：合成完成后可直接播放，查看费用估算
8. **下载文件**：点击下载按钮保存 MP3 文件

## 🎵 音频格式

- **格式**：MP3
- **采样率**：32000 Hz
- **比特率**：128 kbps
- **声道**：双声道

## 📁 文件命名规则

下载的音频文件按以下规则命名：
```
[文本前5个有效字符]-[HH_MM].mp3
```
例如：`真正的危-14_30.mp3`

## 💰 费用计算

- **计费规则**：按字符计费，1汉字=2字符，其他字符=1字符
- **费用标准**：¥3.5/万字符
- **实时显示**：界面会实时显示使用字符数和预估费用

## 🔗 API 接口

使用 MiniMax T2A v2 接口：
- **接口地址**：`https://api.minimaxi.com/v1/t2a_v2`
- **认证方式**：Bearer Token
- **数据格式**：流式 event-stream，hex 编码音频数据
- **输出格式**：支持 hex 编码的流式音频输出

## 🌐 浏览器兼容性

- Chrome 88+
- Firefox 90+
- Safari 14+
- Edge 88+

## 📋 更新日志

### [1.0.3]
- 新增版本记录模态框，可查看历史更新日志
- 页面新增页脚，包含仓库地址、开源协议说明及 vibecoding 声明

### [1.0.2]
- 新增「自定义音色 ID」支持，允许用户输入任意可用音色编号

### [1.0.1] 
- 新增混音功能，最多可选择 4 种音色并配置不同权重进行混合播放
- 优化整体样式与界面一致性

### [1.0.0] 
- 初始版本发布
- 支持 MiniMax T2A 语音合成 API

## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📞 支持

如有问题，请查看 [MiniMax API 文档](https://api.minimaxi.com/document/guides/text-to-speech) 或提交 Issue。

---

**Vibe Coding** - 让技术更有温度
