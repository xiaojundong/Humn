# Humn

白噪音 · 助眠 · 专注

Humn 是一款基于 uni-app 构建的白噪音应用，帮助你在嘈杂环境中集中注意力、放松身心或更快入眠。支持 Android、iOS、H5 和微信小程序。

## 功能

- **18 种预设声音** — 雨声、雷雨、海浪、溪流、篝火、风声、森林鸟鸣、夏夜蝉鸣、风扇、键盘敲击、猫咪呼噜、心跳、咖啡厅、风铃、捏捏乐，以及白噪音、粉红噪音、棕色噪音
- **程序合成噪声** — 白噪音 / 粉红噪音 / 棕色噪音通过 Web Audio API 实时生成，无需音频文件
- **自定义声音** — 支持从本地导入 MP3、WAV、OGG、M4A、AAC 格式的音频文件
- **深色 / 浅色主题** — 可随系统自动切换或手动选择
- **后台播放** — App 端通过 BackgroundAudioManager 支持锁屏和后台持续播放
- **流畅动画** — 播放脉冲、波形动效、弹性弹窗等交互细节

## 项目结构

```
Humn/
├── App.vue                # 应用入口
├── main.js                # 启动配置
├── manifest.json          # uni-app 配置（权限、模块、分发）
├── pages.json             # 页面路由与样式
├── pages/
│   └── index/
│       └── index.vue      # 主页面（声音列表、播放引擎、菜单、导入）
├── static/
│   └── audio/             # 预设音频文件（.mp3）
├── uni.scss               # 全局 SCSS 变量
└── index.html             # H5 入口
```

## 开发环境

- **IDE**: HBuilderX
- **框架**: uni-app (Vue 2 / Vue 3)
- **目标平台**: Android、iOS、H5、微信小程序

## 本地运行

1. 使用 HBuilderX 打开项目根目录
2. 将预设音频文件放入 `static/audio/` 目录（参考 `pages/index/index.vue` 中 `PRESET_SOUNDS` 的文件名列表）
3. 选择运行目标平台，点击运行

## 音频文件

预设声音依赖以下音频文件，需自行放入 `static/audio/`：

| 文件名 | 对应声音 |
|--------|----------|
| `rain.mp3` | 雨声 |
| `thunder.mp3` | 雷雨 |
| `ocean.mp3` | 海浪 |
| `stream.mp3` | 溪流 |
| `campfire.mp3` | 篝火 |
| `wind.mp3` | 风声 |
| `forest.mp3` | 森林鸟鸣 |
| `cicada.mp3` | 夏夜蝉鸣 |
| `fan.mp3` | 风扇 |
| `squeeze.mp3` | 捏捏乐 |
| `keyboard.mp3` | 键盘敲击 |
| `catpurr.mp3` | 猫咪呼噜 |
| `heartbeat.mp3` | 心跳 |
| `coffee.mp3` | 咖啡厅 |
| `bell.mp3` | 风铃 |

> 白噪音、粉红噪音、棕色噪音由程序实时合成，无需音频文件。

## 构建发布

通过 HBuilderX 的原生打包功能进行云打包或本地打包，生成 APK/IPA。

## 技术要点

- **噪声引擎** — 基于 Web Audio API 的 `AudioContext` 实时合成白/粉红/棕色噪音
- **条件编译** — 使用 uni-app 的 `#ifdef APP-PLUS` 等预编译指令区分平台逻辑
- **文件选择** — Android 端通过原生 `Intent.ACTION_OPEN_DOCUMENT` 调用系统文件选择器
- **持久化** — 自定义声音数据通过 `uni.setStorageSync` 存储在本地
