# pymss-colab

一键在 Google Colab 上运行 [pymss](https://github.com/pymss-project/pymss)（Python Music Source Separation），自动安装、启动 WebUI 服务器，并通过 Cloudflare Quick Tunnel 暴露公网临时域名。

**[Open in Colab](https://colab.research.google.com/github/uwuAitry/pymss-colab/blob/master/pymss_colab.ipynb)**

## Features

- 一键运行：单个 cell 即可完成所有配置
- GPU 加速：自动检测 CUDA GPU
- 公网访问：通过 Cloudflare Quick Tunnel 获得临时 `*.trycloudflare.com` 域名
- WebUI 界面：浏览器操作，上传音频即可分离人声、鼓、贝斯等
- 模型按需加载：首次使用时自动下载模型

## Quick Start

1. 打开 [Google Colab](https://colab.research.google.com/)
2. 上传 `pymss_colab.ipynb` 或通过 URL 导入
3. **Runtime > Change runtime type > T4 GPU**
4. 运行唯一的代码 cell
5. 等待输出中出现 `https://*.trycloudflare.com` 公网 URL
6. 在浏览器中打开 WebUI 链接

## Requirements

- Google Colab 免费账号（带 GPU runtime）
- 浏览器

## What It Does

```
GPU Check → Install pymss → Download cloudflared → Start Server → Public URL
```

| Step | Description |
|------|-------------|
| GPU Check | 验证 CUDA 可用，显示 GPU 型号和显存 |
| Install | `pip install pymss[server]` |
| cloudflared | 下载 Cloudflare Quick Tunnel 二进制 |
| Server | `pymss serve --webui` 在后台运行 |
| Tunnel | 暴露临时公网 URL |

## Usage

### WebUI
打开公网 URL 后的 WebUI 链接即可使用：
- 浏览/搜索模型目录
- 点击加载模型（首次自动下载）
- 上传音频文件
- 点击分离
- 下载分离后的音轨

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
A: 默认下载源是 ModelScope。可以在 WebUI 中切换到 Hugging Face。

**Q: 公网 URL 打不开？**
A: Cloudflare Quick Tunnel 是临时的，session 结束后会失效。重新运行 notebook 即可。

## License

MIT

## Credits

- [pymss](https://github.com/pymss-project/pymss) - Python Music Source Separation
- [Cloudflare cloudflared](https://github.com/cloudflare/cloudflared) - Quick Tunnel
