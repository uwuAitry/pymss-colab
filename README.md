# pymss-colab

在 Google Colab 上一键运行 [pymss](https://github.com/pymss-project/pymss)（Python Music Source Separation），自动安装依赖、启动 WebUI/ComfyUI 服务器，并通过 Cloudflare Quick Tunnel 暴露公网临时域名。

**[Open in Colab](https://colab.research.google.com/github/uwuAitry/pymss-colab/blob/master/pymss_colab.ipynb)**

## Notebook 结构

Notebook 共 4 步（Cell），全部以 Colab form 模式呈现：

| Step | Cell | 说明 |
|------|------|------|
| 1 | Install dependencies | 检测 GPU → `pip install pymss[server]` → 下载 cloudflared → 安装 ComfyUI + comfy-mss + ComfyUI Manager |
| 2 | Mount Google Drive (optional) | 挂载 Drive 以读取模型配置文件 |
| 3 | Configure & download models | 填写模型 ID 或读取 Drive 配置，支持 HuggingFace / ModelScope 下载 |
| 4 | Start server + tunnel | 选择 webui / comfyui 模式，启动服务器 + cloudflared 隧道 |

### Step 1 — Install dependencies
- 自动检测 CUDA GPU 并显示型号/显存
- 安装 `pymss[server]`（含 WebUI + REST API）
- 下载 cloudflared 二进制
- 克隆 ComfyUI + comfy-mss + ComfyUI Manager + comfyui-frontend-package
- 预创建 Manager `config.ini` 设置 `security_level = weak`（绕过 `--listen 0.0.0.0` 时的 loopback 限制）

### Step 2 — Mount Google Drive (optional)
- 挂载到 `/content/drive`
- 模型配置文件路径：`/content/drive/MyDrive/pymss/models.txt`（每行一个模型 ID，`#` 开头为注释）

### Step 3 — Configure & download models
- **MODELS** 表单：直接填写模型 ID（每行一个），留空则读取 Drive 配置
- **MODEL_SOURCE**：`huggingface` 或 `modelscope`
- **MODEL_DIR**：默认 `/tmp/pymss_models`
- 已下载的模型会跳过，失败的模型会列出错误信息

### Step 4 — Start server + tunnel
- **MODE**：`webui`（pymss WebUI）或 `comfyui`（ComfyUI + comfy-mss）
- **PORT**：默认 8188
- 服务器日志实时输出到 cell
- cloudflared 自动获取 `*.trycloudflare.com` 临时域名
- 点击 ■ 停止服务器和隧道

## Features

- 一键运行：4 个 cell 即可完成全部配置
- GPU 加速：自动检测 CUDA GPU
- 公网访问：通过 Cloudflare Quick Tunnel 获得临时 `*.trycloudflare.com` 域名
- 双模式：pymss WebUI 或 ComfyUI + comfy-mss
- ComfyUI Manager：支持在 ComfyUI 内安装/管理自定义节点（`security_level = weak`，适配 `--listen 0.0.0.0`）
- 模型预下载：支持从 HuggingFace / ModelScope 下载，或通过 Google Drive 配置
- 服务器日志实时输出

## Quick Start

1. 打开 [Google Colab](https://colab.research.google.com/)
2. 上传 `pymss_colab.ipynb` 或通过上方链接直接打开
3. **Runtime > Change runtime type > T4 GPU**
4. 按顺序运行 4 个 Cell
5. 等待 Step 4 输出 `https://*.trycloudflare.com` 公网 URL
6. 在浏览器中打开 WebUI 链接

## Usage

### WebUI
打开公网 URL 后的 WebUI 链接即可使用：
- 浏览/搜索模型目录
- 点击加载模型（首次自动下载）
- 上传音频文件
- 点击分离
- 下载分离后的音轨

> ⚠️ **Known Issue:** WebUI 的文件上传/下载功能在 Cloudflare 隧道下可能存在兼容性问题。上传音频后分离结果可能无法正常返回。此问题可能与 pymss WebUI 前端本身的实现有关，非本 notebook 导致。建议使用 REST API 或 Python API 作为替代方案。

### ComfyUI
- 打开公网 URL 后直接进入 ComfyUI 界面
- 加载/构建使用 MSS Separate 节点的工作流
- 支持工作流保存和复用

### Python API

```python
from pymss import MSSeparator

with MSSeparator.from_model_name(
    "bs_roformer_voc_hyperacev2",
    download=True,
    device="auto",
    output_format="wav",
    store_dirs="results",
) as sep:
    sep.process_folder("/path/to/audio")
```

### REST API

```bash
# 列出模型
curl https://<your-url>.trycloudflare.com/v1/catalog/models

# 加载模型
curl -X POST https://<your-url>.trycloudflare.com/v1/models/load \
  -H 'Content-Type: application/json' \
  -d '{"model": "bs_roformer_voc_hyperacev2"}'

# 分离音频
curl -X POST "https://<your-url>.trycloudflare.com/v1/audio/separations?model=bs_roformer_voc_hyperacev2&format=pcm_f32le&sample_rate=44100&channels=2&response_format=zip&output_audio_format=wav" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @input.wav -o result.zip
```

## Supported Models

pymss 支持多种模型，包括但不限于：

| Model | Type | Target |
|-------|------|--------|
| bs_roformer_voc_hyperacev2 | bs_roformer | Vocals |
| htdemucs | htdemucs | Vocals, Drums, Bass, Other |
| mdx23c | mdx23c | Vocals |
| mel_band_roformer | mel_band_roformer | Vocals |
| VR 系列 | vr | Vocals, Instrumental |

详见 [pymss 模型目录](https://github.com/pymss-project/pymss)。

## Troubleshooting

**Q: 没有 GPU 怎么办？**
A: Colab 免费版可能分配不到 GPU，尝试 Runtime > Disconnect and delete runtime，然后重新连接。

**Q: 模型下载失败？**
A: 默认下载源是 ModelScope。可以在 Step 3 中切换 MODEL_SOURCE 到 HuggingFace。

**Q: 公网 URL 打不开？**
A: Cloudflare Quick Tunnel 是临时的，session 结束后会失效。重新运行 Step 4 即可。

**Q: WebUI 上传文件无反应？**
A: 这是已知问题，可能与 pymss WebUI 前端在 Cloudflare 隧道下的兼容性有关。建议使用 REST API 或 Python API 进行音频分离。

## Known Issues

- **WebUI 文件上传/下载：** 在 Cloudflare 隧道下，WebUI 的文件上传和下载功能可能无法正常工作。上传音频后点击分离可能无响应，或分离结果无法下载。此问题可能与 pymss WebUI 前端的音频处理逻辑（浏览器端 PCM 编码 + octet-stream 上传）在隧道环境下的兼容性有关。建议使用 REST API 或 Python API 替代。
- **ComfyUI 403 错误：** 已通过 `--enable-cors-header` 解决。

## License

MIT

## Credits

- [pymss](https://github.com/pymss-project/pymss) — Python Music Source Separation，核心分离引擎
- [Cloudflare cloudflared](https://github.com/cloudflare/cloudflared) — Quick Tunnel 公网暴露
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI) — 节点式工作流界面
- [comfy-mss](https://github.com/pymss-project/comfy-mss) — ComfyUI MSS 分离节点
