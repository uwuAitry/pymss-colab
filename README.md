# pymss-colab

在 Google Colab 上一键运行 [pymss](https://github.com/pymss-project/pymss)（Python Music Source Separation），自动安装依赖、启动 WebUI/ComfyUI 服务器，并通过 Cloudflare Quick Tunnel 暴露公网临时域名。另提供 AutoDL 一键部署脚本（ComfyUI + comfy-mss + UniverSR）。

**[Open in Colab](https://colab.research.google.com/github/uwuAitry/pymss-colab/blob/master/pymss_colab.ipynb)**

## 两种运行方式

| 方式 | 入口 | 适用场景 |
|------|------|----------|
| Google Colab（下文原有文档） | `pymss_colab.ipynb` | 免费/T4 GPU，Cloudflare 临时隧道 |
| AutoDL（见 [AutoDL 部署](#autodl-部署comfyui--comfy-mss--universr)） | `autodl_setup.sh` + `start_comfyui.sh` | 独占 GPU 实例，固定 6006 端口自定义服务，数据盘持久化 |

### 仓库文件

| 文件 | 用途 |
|------|------|
| `pymss_colab.ipynb` | Google Colab 一键运行 notebook（原有方案） |
| `autodl_setup.sh` | AutoDL 一键安装脚本（幂等、fail-fast，可重复执行） |
| `start_comfyui.sh` | AutoDL 启动 ComfyUI（127.0.0.1:6006，无隧道） |
| `comfy-mss/ensemble_all_in.json` | comfy-mss 工作流示例 |
| `custom_nodes/comfy-universr/` | UniverSR 音频超分辨率 ComfyUI 节点（详见其 README） |

## AutoDL 部署（ComfyUI + comfy-mss + UniverSR）

### Quick Start

在 AutoDL 实例（选带 GPU 的镜像）终端中执行：

```bash
git clone https://github.com/uwuAitry/pymss-colab.git pymss-for-colab
cd pymss-for-colab
bash autodl_setup.sh    # 安装 ComfyUI + 自定义节点 + UniverSR（已装的部分自动跳过）
bash start_comfyui.sh   # 启动 ComfyUI，监听 127.0.0.1:6006
# 然后：AutoDL 控制台 → 自定义服务 → 代理端口 6006 → 浏览器打开
```

不需要 cloudflared：AutoDL 的「自定义服务」直接代理实例的 6006 端口。

### 环境变量（均可覆盖）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PYMSS_ROOT` | `/root/autodl-tmp/pymss` | 持久化根目录（HF 缓存、模型、临时文件） |
| `COMFYUI_DIR` | `/root/autodl-tmp/ComfyUI` | ComfyUI 安装目录 |
| `PORT` | `6006` | 启动脚本监听端口 |
| `LISTEN` | `127.0.0.1` | 监听地址 |

### 持久化路径

- `/root/autodl-tmp` 是 AutoDL 数据盘，实例关机/重置后数据保留（系统盘会重置）
- ComfyUI 与 HuggingFace 缓存（`HF_HOME=$PYMSS_ROOT/hf`，含 UniverSR 权重）都在数据盘；实例重置后重跑 `autodl_setup.sh` 即可恢复，已下载内容自动跳过
- `start_comfyui.sh` 启动时同样导出 `HF_HOME`，保证节点运行时下载的模型也落在持久盘

### ComfyUI Manager 注意事项

- 安装脚本预写 `ComfyUI/user/__manager/config.ini`（当前 Manager 的配置路径），设置 `security_level = weak`
- 原因：AutoDL 自定义服务代理转发的请求不是 loopback 来源，Manager 默认安全级别会拦截部分操作（如在线安装节点）
- ⚠️ `weak` 会放宽 Manager 的安全校验，仅建议在个人实例上使用

### UniverSR 节点（音频超分辨率 ≠ 人声分离）

**UniverSR 不是 MSS 源分离模型**：它是音频超分辨率/带宽扩展模型，把 8/12/16/24 kHz 带宽的全混音提升到 48 kHz（vocoder-free flow matching，ICASSP 2026）。它输入 AUDIO、输出**一路**增强后的 AUDIO，不会输出人声/伴奏等音轨。

- 源分离用 comfy-mss 的节点；带宽扩展用 `UniverSR Enhance` 节点，两者可串联（如 `Load Audio → UniverSR Enhance → MSS Separate`）
- 节点参数与用法见 [`custom_nodes/comfy-universr/README.md`](custom_nodes/comfy-universr/README.md)
- 模型首次运行时从 HuggingFace 下载（`woongzip1/universr-audio` / `woongzip1/universr-speech`），缓存于 `HF_HOME`

### UniverSR 官方链接

- 代码（MIT）：https://github.com/woongzip1/UniverSR
- 论文：https://arxiv.org/abs/2510.00771
- 模型：https://huggingface.co/woongzip1/universr-audio 、https://huggingface.co/woongzip1/universr-speech

## Notebook 结构

Notebook 共 3 步（Cell），全部以 Colab form 模式呈现：

| Step | Cell | 说明 |
|------|------|------|
| 1 | Install dependencies | 检测 GPU → `pip install pymss[server]` → 下载 cloudflared → 安装 ComfyUI + comfy-mss + ComfyUI Manager |
| 2 | Mount Google Drive (persistence) | 挂载 Drive 并把 models / output / input / user / HF 缓存重定向到 Drive，断线不丢 |
| 3 | Start server + tunnel | 选择 webui / comfyui 模式，启动服务器 + cloudflared 隧道（ComfyUI 走 Drive 路径） |

### Step 1 — Install dependencies
- 自动检测 CUDA GPU 并显示型号/显存
- 安装 `pymss[server]`（含 WebUI + REST API）
- 下载 cloudflared 二进制
- 克隆 ComfyUI + comfy-mss + ComfyUI Manager + comfyui-frontend-package
- 安装 UniverSR 包，并把 `comfy-universr` 节点源码内联写入 `custom_nodes/`（`custom_nodes/` 未进仓库，所以不靠 git clone）
- 克隆 `comfyui-output-swap`（连线替换的前端小工具，上游已带 `web/dist/`，无需 bun 构建）
- Manager `config.ini` 由 Step 3 写入（`security_level = weak`），因为它的位置取决于 Step 2 是否开启持久化

### Step 2 — Mount Google Drive (persistence)
- 挂载到 `/content/drive`，持久化根目录 `DRIVE_ROOT`（默认 `/content/drive/MyDrive/pymss`）

| Drive 子目录 | 内容 |
|------|------|
| `models/` | ComfyUI 模型（checkpoints / vae / loras / diffusion_models …） |
| `output/` | 分离结果，在 Google Drive 网页直接下载 |
| `input/` | 上传到 ComfyUI 的音频 |
| `user/` | ComfyUI 设置 + Manager `config.ini`（含 `security_level`） |
| `hf/` | HuggingFace / ModelScope 权重缓存（comfy-mss、UniverSR） |

- 同时设置 `HF_HOME` / `HUGGINGFACE_HUB_CACHE` / `MODELSCOPE_CACHE` 指向 `hf/`，与 `start_comfyui.sh` 的约定一致
- 关闭 `ENABLE_DRIVE_PERSIST` 则回到临时路径（每次重连都要重新下载模型、输出不保留）
- ⚠️ Drive 占用你的 Google 配额，且经 FUSE 写入较慢；一次性任务可关掉持久化

### Step 3 — Start server + tunnel
- **MODE**：`webui`（pymss WebUI）或 `comfyui`（ComfyUI + comfy-mss）
- ComfyUI 以 `--models-directory` / `--output-directory` / `--input-directory` / `--user-directory`
  指向 Step 2 的 Drive 路径（关闭持久化时用 ComfyUI 默认路径）
- **PORT**：默认 8188
- 服务器日志实时输出到 cell
- cloudflared 自动获取 `*.trycloudflare.com` 临时域名
- 点击 ■ 停止服务器和隧道

## Features

- 一键运行：3 个 cell 即可完成全部配置
- GPU 加速：自动检测 CUDA GPU
- 输出持久化：分离结果写入 Drive `output/`，在 Google Drive 网页直接下载
- 公网访问：通过 Cloudflare Quick Tunnel 获得临时 `*.trycloudflare.com` 域名
- 双模式：pymss WebUI 或 ComfyUI + comfy-mss
- ComfyUI Manager：支持在 ComfyUI 内安装/管理自定义节点（`security_level = weak`，适配 `--listen 0.0.0.0`）
- 持久化：Step 2 把模型、输出、上传音频、配置与 HF 缓存全部落到 Google Drive，断线重连不丢
- 模型持久化：ComfyUI 模型与 HF/ModelScope 权重缓存均落 Drive，重连不重新下载
- 服务器日志实时输出
- UniverSR 超分节点：`UniverSR Enhance`（带宽扩展，非源分离），可与 MSS Separate 串联
- Output Swap：拖一个节点的输出到另一个节点的输出口，接管它下游所有连线
- ⚠️ UniverSR 会额外占用显存，首次运行还要下载 flow-matching 权重；只用源分离的话别加这个节点

## Quick Start

1. 打开 [Google Colab](https://colab.research.google.com/)
2. 上传 `pymss_colab.ipynb` 或通过上方链接直接打开
3. **Runtime > Change runtime type > T4 GPU**
4. 按顺序运行 3 个 Cell
5. 等待 Step 3 输出 `https://*.trycloudflare.com` 公网 URL
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
- 另装 `UniverSR Enhance` 节点：把全混音从 8/12/16/24 kHz 带宽升到 48 kHz（带宽扩展，**不是**源分离，只输出一路）。可与 MSS 串联：`Load Audio → UniverSR Enhance → MSS Separate`
- 装 `comfyui-output-swap`：把 A 节点的输出拖到 B 节点的输出口上，A 会接管 B 下游的全部连线（省得满画布找下游节点）。拖动时 B 已有匹配空输入则顺带把 A 插进链路，按住 Alt 可跳过；开关在 Settings → Touch Tools → Output Swap。需要前端 ≥ 1.43，否则自动不生效（不报错）

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
A: 权重都是从 HuggingFace / ModelScope 官方源实时下载的（缓存到 Drive `hf/`）。先确认 Drive 配额未满，再重跑 Step 3。

**Q: 公网 URL 打不开？**
A: Cloudflare Quick Tunnel 是临时的，session 结束后会失效。重新运行 Step 3 即可。

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
- [UniverSR](https://github.com/woongzip1/UniverSR) — 音频超分辨率（带宽扩展）模型，ICASSP 2026，MIT
