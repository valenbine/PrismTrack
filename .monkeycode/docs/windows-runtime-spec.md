# PrismTrack Windows Runtime Spec

## 目标

为 `PrismTrack` 建立一套可复制、可发布、可诊断的 Windows 本地运行时方案，优先参考已被实物验证的 `SpleeterGUI 2.9.4` 运行时组合，而不是依赖用户系统中已有的 Python、Spleeter 或 ffmpeg。

## 设计原则

1. Windows 版必须使用固定版本运行时，不依赖用户机器上的全局 Python 环境。
2. Windows 版必须优先使用应用目录中的本地运行时，不依赖系统 PATH 中的 `ffmpeg`、`ffprobe` 或 `spleeter`。
3. 模型权重默认不随安装包分发，使用 `Lazy Model Fetch` 在首次使用时下载。
4. 所有健康检查和错误提示必须尽量指向明确根因，例如缺少 VC Runtime、模型下载失败、CPU 不支持 AVX、TensorFlow 导入失败。

## 参考基线

基于 `SpleeterGUI 2.9.4.0` 发布包实物分析，优先参考以下基线：

- Python: `3.7`
- Spleeter: `2.3.1`
- TensorFlow: `2.5.0`
- NumPy: `1.17.3`
- SciPy: `1.4.1`
- Librosa: `0.8.0`
- ffmpeg: 本地携带 `ffmpeg.exe`、`ffprobe.exe`、`ffplay.exe`
- 模型目录: `pretrained_models/` 预创建但默认空目录

说明：

- 上述版本组合不是理论上的最新组合，而是已经出现在可工作的 Windows 发布包中的实物组合。
- 如果后续升级版本，必须先建立新的 Windows 验证矩阵，不能直接无条件替换。

## 目标目录结构

Windows 发行目录建议采用如下结构：

```text
PrismTrack/
  PrismTrack.exe
  server.js
  package.json
  index.html
  styles.css
  src/
  scripts/
    spleeter_separate.py
  python/
    python.exe
    python37.dll
    Lib/
    Scripts/
  ffmpeg.exe
  ffprobe.exe
  ffplay.exe
  pretrained_models/
  .runtime/
    uploads/
    prismtrack-stems/
```

说明：

- `PrismTrack.exe` 可以是桌面宿主，也可以是启动器。
- `python/` 是完整的内置 Python 运行时。
- `pretrained_models/` 为应用级默认模型缓存目录。
- `.runtime/` 为任务临时输出目录。

## 本地运行时解析顺序

Windows 版后端在解析运行时命令时，建议按以下优先级：

1. 应用根目录下的 `python/python.exe`
2. 明确配置的 `SPLEETER_PYTHON`
3. 明确配置的 `SPLEETER`
4. 系统 `spleeter`

`ffmpeg` 与 `ffprobe` 解析顺序建议为：

1. 应用根目录下的 `ffmpeg.exe` / `ffprobe.exe`
2. 明确配置的环境变量
3. 系统 PATH 中的 `ffmpeg` / `ffprobe`

## 模型缓存策略

采用 `Lazy Model Fetch`：

1. 安装包不携带 `2stems`、`4stems`、`5stems` 权重。
2. 首次选择某个模型时，检查本地缓存目录是否已存在且完整。
3. 若不存在，则开始下载，并在现有状态区显示：
   - 当前模型名
   - 下载进度
   - 已下载字节数与总字节数
   - 失败原因
4. 下载完成后，执行完整性检查。
5. 检查失败时，删除损坏缓存并提示用户重试。
6. 下载中的归档使用 `.tar.gz.part` 稳定文件名，支持中断后通过 HTTP Range 断点续传。
7. Windows 下如果 Node `fetch` 因证书链问题失败，回退到系统 `curl.exe` 下载，并使用 `--continue-at -` 保留续传能力。

推荐缓存位置：

1. Web 开发默认使用应用目录下的 `pretrained_models/`
2. Windows 桌面版使用 Electron `userData` 下的缓存路径，例如：
   - `%APPDATA%/prismtrack-spleeter/pretrained_models/`
3. Windows 桌面版下载临时归档位于：
   - `%APPDATA%/prismtrack-spleeter/.runtime/model-downloads/`

## 桌面启动协议

桌面宿主层不直接等待深度 `/api/health`。启动等待协议为：

1. 启动本地 `server.js`
2. 优先请求 `GET /api/ready`
3. 如果 `/api/ready` 不可用，则回退请求首页 `/`
4. 首页可返回 `200` 时即可打开窗口
5. 深度 `/api/health` 仅作为后台运行时诊断，不阻塞窗口创建

## 启动前健康检查与诊断

Windows 版建议在应用启动时执行以下检查：

1. 本地 `python/python.exe` 是否存在
2. 本地 `ffmpeg.exe` 与 `ffprobe.exe` 是否存在
3. Python 是否能成功导入：
   - `spleeter`
   - `tensorflow`
   - `numpy`
4. TensorFlow 是否能初始化基础运行时
5. CPU 是否支持 AVX
6. VC Runtime 是否已安装
7. 所选模型权重是否存在且完整

其中 1、2 和包装脚本存在性属于启动前硬校验。Python/Spleeter/TensorFlow 等深度诊断可能耗时，应通过 `/api/health` 后台执行并写入桌面日志。

桌面日志路径：

```text
%APPDATA%/prismtrack-spleeter/logs/desktop.log
```

## 错误分类与提示规范

必须将常见故障拆分为明确类别，而不是统一显示 “Spleeter 不可用”。

建议错误类别：

1. `missing_python_runtime`
2. `missing_ffmpeg_runtime`
3. `missing_vc_runtime`
4. `tensorflow_import_failed`
5. `cpu_avx_unsupported`
6. `model_not_downloaded`
7. `model_download_failed`
8. `model_cache_corrupted`
9. `separation_failed`

## 后端改造要求

为兼容上述 Windows 运行时布局，`server.js` 后续应满足：

1. 自动探测应用根目录下的本地 Python 与 ffmpeg
2. `health` 接口输出更细粒度状态
3. 允许指定模型缓存目录
4. 支持模型下载进度事件
5. 出现模型损坏时可自动清理并重试

## 当前项目与目标状态差异

当前 `PrismTrack` Windows 状态：

- CI 从 `SpleeterGUI 2.9.4.0` 下载并注入 Python 与 ffmpeg 运行时
- 桌面主进程显式传入 `SPLEETER_PYTHON`、`FFMPEG`、`FFPROBE`、`SPLEETER_WRAPPER`
- 模型按需下载到用户数据目录
- 支持模型下载进度、失败原因、断点续传与 Windows curl TLS 兜底
- 桌面启动日志记录路径解析、服务输出、运行时诊断和模型下载错误

## 实施顺序建议

1. 继续补充 Windows 图标和安装体验细节
2. 按需增加 AVX 与 VC Runtime 更细粒度诊断
3. 建立新的 Windows 运行时升级验证矩阵后再升级 Python/Spleeter/TensorFlow
