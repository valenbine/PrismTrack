# PrismTrack

PrismTrack 是一个基于 Spleeter 的音乐分轨项目，提供：

- Web 版分轨体验
- Windows 桌面安装包构建流程
- 多模型音轨分离与结果下载能力

当前仓库主要围绕 `PrismTrack-Spleeter/` 目录展开，包含前端页面、Node.js 后端、Spleeter 集成，以及 Electron Windows 桌面封装配置。

## 核心能力

- 上传常见音频文件并执行分轨
- 支持 `spleeter:2stems`、`spleeter:4stems`、`spleeter:5stems`
- 支持试听、静音、单轨播放、音量控制
- 支持单独下载音轨或打包下载全部结果
- 支持模型按需下载、断点续传和 Windows 下载 TLS 兜底
- 提供 GitHub Actions Windows 安装包构建流程

## 仓库结构

- `PrismTrack-Spleeter/`: 主应用目录
- `.github/workflows/`: GitHub Actions Windows 打包工作流
- `.monkeycode/`: 项目记忆、规范和规格文档

## 快速开始

Web 应用入口说明见：

- `PrismTrack-Spleeter/README.md`

在本地运行主应用：

```bash
# Install dependencies
cd PrismTrack-Spleeter && npm install

# Start the app
npm start
```

默认访问地址：`http://127.0.0.1:8000/`

## Windows 构建

仓库已包含 Windows 桌面应用打包 workflow，入口位置：

- `.github/workflows/build-windows.yml`

打包流程会在 Windows runner 内下载 `SpleeterGUI 2.9.4.0` 运行时基线，注入 `python/` 与 `ffmpeg.exe`、`ffprobe.exe`、`ffplay.exe` 后生成 NSIS 安装包。模型权重不内置，首次使用时下载到用户数据目录。

## 当前状态

当前分支已完成：

- Windows Electron 桌面宿主
- SpleeterGUI 运行时基线注入
- Lazy Model Fetch 与断点续传
- Windows 启动日志与运行时诊断
- GitHub Actions Windows 安装包产物构建

如果你要快速了解具体应用实现，请优先阅读：

- `PrismTrack-Spleeter/README.md`
- `PrismTrack-Spleeter/server.js`
- `PrismTrack-Spleeter/src/main.js`
