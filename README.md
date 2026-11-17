# 视频转文本 Browser Whisper ASR

**在浏览器里跑 Whisper 语音识别，文件不上传、不调接口、零服务器成本。**

在线体验：<https://www.cqqzx.com/tools/video-to-text.html?utm_source=github&utm_medium=readme&utm_campaign=demo>（完整版：mp4 抽音轨、SRT 导出、长音频云端引擎）

本仓库只开源其中的核心转写模块，约 150 行，无构建步骤，直接静态托管即可运行。下文所有性能数据都来自这个线上版的真实测试。

---

## 它解决什么问题

所有在线转写服务都要把文件传到对方服务器：快，但文件离开了你的设备，而且免费额度用完就收费。

Whisper 是 OpenAI 开源的语音识别模型，经过量化后只有 40–250MB，完全可以塞进浏览器。配合 [transformers.js](https://github.com/xenova/transformers.js)（ONNX Runtime Web 的封装），推理过程就是一段 WASM。

于是有了另一种取舍：**用你电脑的 CPU 换隐私和免费**。

| | 本项目（浏览器本地） | 常见在线转写 |
|---|---|---|
| 文件去向 | 不出本机 | 上传到对方服务器 |
| 费用 | 0 | 免费额度 + 超出充值 |
| 速度 | 音频时长的 4–5 倍 | 数十秒 |
| 隐私 | 最强 | 取决于服务商 |

### 本仓库与线上版的分工

这里刻意**只开源核心转写模块**——Worker 封装、音频解码、SRT 生成，不含站点样式、统计和业务代码。

[线上完整版](https://www.cqqzx.com/tools/video-to-text.html?utm_source=github&utm_medium=readme&utm_campaign=full) 在此之上多做了三件事：mp4/mov 先用 ffmpeg.wasm 抽音轨；超过 2 分钟可切到本地/云端两条路；导出 SRT 后能直接跳到字幕压制、切片和脚本生成。想直接解决问题用线上版，想集成到自己的项目里看下面的源码。

## 性能实测

数据取自 [线上版](https://www.cqqzx.com/tools/video-to-text.html?utm_source=github&utm_medium=readme&utm_campaign=benchmark) 内置提示与 Edge + CDP 端到端测试（i5-1135G7，单线程 WASM，whisper-tiny）。

| 音频时长 | 识别耗时 | 建议 |
|---|---|---|
| 15 秒 | 约 2 分钟 | 体验最好 |
| 30 秒 | 约 3 分钟 | 推荐 |
| 1 分钟 | 约 5 分钟 | 可接受 |
| 2 分钟 | 约 10 分钟 | 接近上限 |
| 5 分钟以上 | 20 分钟以上 | 不建议，改用云端 API |

经验公式：**总耗时 ≈ 45 秒准备时间 + 音频时长 × 4~5 倍**。

模型选择：

| 模型 | 体积 | 中文表现 | 适用 |
|---|---|---|---|
| whisper-tiny | 40MB | 一般，长句易漏字 | 快速出稿 |
| whisper-base | 80MB | 较好 | 日常首选 |
| whisper-small | 250MB | 好 | 正式内容 |

## 快速开始

```bash
git clone https://github.com/<your-name>/browser-whisper-asr.git
cd browser-whisper-asr
python -m http.server 8080
# 打开 http://localhost:8080
```

> 必须用 HTTP 打开，`file://` 下 module worker 会被 CORS 拦掉。

### 模型从哪儿下

默认走 `huggingface.co`。国内网络访问不了的话，构造时换镜像：

```js
const asr = createASR({ remoteHost: 'https://hf-mirror.com' });
```

但要先确认镜像给你的域名回了 `Access-Control-Allow-Origin`——有些镜像只对自己的域名放开，浏览器会直接报 `Failed to fetch`，而 curl 测试是通的，很容易误判。

最稳的方案是**自己托管模型**：把 `Xenova/whisper-base` 的 `config.json`、`tokenizer.json`、`preprocessor_config.json`、`onnx/*.onnx` 放到同域目录，然后：

```js
env.allowLocalModels = true;
env.localModelPath = '/models/';      // 只下过一次，之后不再依赖外网
```

同理，`transformers.min.js` 和 `ort-wasm-*.wasm` 也建议自托管（CDN 在部分网络下会整段超时）。

## 用法

```js
import { createASR } from './src/asr.js';
import { decodeToMono16k } from './src/audio.js';

const audio = await decodeToMono16k(file);        // → 16kHz 单声道 Float32Array
const asr = createASR({ model: 'Xenova/whisper-base', language: 'chinese' });
await asr.load('Xenova/whisper-base', (p) => console.log(p.status, p.progress));
const res = await asr.transcribe(audio, { withTimestamps: true });

console.log(res.text);      // 全文
console.log(res.chunks);    // [{ text, timestamp: [start, end] }, ...]
```

## 三个真正会卡住你的坑

这三条都是实际调试出来的，不是文档里抄的。

### 1. Blob Worker 里的 `import('/tools/xx.js')` 会 404

Worker 是用 `URL.createObjectURL(new Blob([src]))` 建的，它的 **base URL 是 `blob:`**。此时：

```js
import('/tools/assets/transformers/transformers.min.js');  // ✗ Failed to resolve module specifier
```

浏览器把 `/tools/...` 当成裸模块名去查 import map，报的是「无法解析模块名」，而不是 404，很容易往错误方向排查。

解法：在 Worker 里拼绝对地址。

```js
const base = location.origin;               // 传入 worker 前算好也可以
import(`${base}/tools/assets/transformers/transformers.min.js`);   // ✓
```

本仓库的 `src/asr.js` 直接把 CDN 绝对 URL 拼进 Worker 源码，同一道理。

### 2. 不能从第三方 CDN 直接 new Worker

```js
new Worker('https://cdn.jsdelivr.net/npm/@xenova/transformers@2/dist/worker.js');  // ✗ 
```

跨源 Worker 会被同源策略拒绝。要么把 worker.js 放到自己域名下，要么像本项目一样用 Blob + 绝对 URL import。

### 3. `numThreads > 1` 需要跨源隔离，静态托管上基本拿不到

多线程 WASM 依赖 `SharedArrayBuffer`，页面必须带这两个响应头：

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

GitHub Pages、大部分虚拟主机都不给配，所以本项目固定 `numThreads = 1`。这也是「为什么慢」的根本原因——单线程跑 encoder，没别的办法。

## 视频文件怎么办

`decodeAudioData` 只认纯音频容器。mp4 / mov 需要先抽音轨：

```js
// ffmpeg.wasm（约 30MB，首次加载慢）
await ffmpeg.exec(['-i', 'in.mp4', '-vn', '-ac', '1', '-ar', '16000', '-c:a', 'pcm_s16le', 'out.wav']);
const pcm = await ffmpeg.readFile('out.wav');
```

注意 `@ffmpeg/core-st` 的最新版本是 0.11.1，**`0.12.6` 不存在**（网上不少示例写错，会一直 404）。用 `@ffmpeg/core@0.12.6`。

线上完整版对视频文件走的就是这条路，音频文件则优先走原生解码（快得多），[在线版入口](https://www.cqqzx.com/tools/video-to-text.html?utm_source=github&utm_medium=readme&utm_campaign=video)。

## 已知限制

- 单线程推理，长音频体验差；超过 2 分钟建议切片或改用云端 API
- 模型首次下载 40–250MB，移动网络下建议先选 tiny
- 手机上跑 small 模型容易因内存被系统回收页面
- 方言、多人重叠说话准确率明显下降

## 目录

```
.
├── index.html        # 可直接打开的 Demo
├── src/
│   ├── asr.js        # Worker 封装 + 模型加载 + SRT 生成
│   └── audio.js      # 解码 + 降采样到 16kHz 单声道
└── docs/
    └── why-slow.md   # 为什么 15 秒音频要跑 2 分钟
```

## 维护方

由 [cqqzx.com](https://www.cqqzx.com/?utm_source=github&utm_medium=readme&utm_campaign=org) 维护，站点上还有视频切片、字幕压制、GIF 提取等纯前端工具。遇到识别异常可以先看 [线上版的常见问题](https://www.cqqzx.com/tools/video-to-text.html?utm_source=github&utm_medium=readme&utm_campaign=faq#faq)，里面列了实测踩过的坑。

## License

MIT
