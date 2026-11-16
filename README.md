# MOSS-Image 本地部署 · 内网穿透 · 云端调用方案

> 一份可复用的技术方案：在**本地电脑**（装有 NVIDIA GPU）部署开源文生图模型
> MOSS-Image，通过**内网穿透**把本地服务暴露到公网，供**云端 AI / 程序**按需调用生成图片。
>
> **本仓库只包含方案思路与部署步骤，不包含任何真实部署环境、密钥或本地文件。**
>
> 📄 **API 调用用法示意请见 [API_USAGE.md](API_USAGE.md)**

---

## 一、方案概述

```
云端 AI / 程序
      │  HTTPS POST /generate
      ▼
公网域名 (内网穿透, 如 cpolar / ngrok)
      │
      ▼
本地守护进程 (按需启动调度器, Guard)
      │  │
      │  ├── 有请求 → 自动拉起文生图主服务
      │  └── 闲置超时 → 自动停止主服务, 释放显存
      ▼
文生图主服务 (MOSS-Image + CUDA GPU)
      │
      ▼
返回 PNG 图片
```

**核心亮点：**
- 🎨 **本地 GPU 出图**：免费、无 API 计费、数据不出本机。
- 🔄 **按需启动**：平时主服务停止（不占显存），有请求才启动；闲置自动停，省资源。
- 🌐 **公网可调**：内网穿透把本地服务暴露成公网 HTTPS 地址，云端可调用。
- 🔑 **鉴权保护**：API Key 校验，防止公网被他人盗用。

---

## 二、环境要求

| 组件 | 要求 |
|---|---|
| GPU | NVIDIA，显存 ≥ 12GB（MOSS-Image 约 12.9GB） |
| 驱动 | CUDA 支持（NVIDIA Driver 新版） |
| Python | ≥ 3.10（3.11 推荐） |
| 系统 | Windows / Linux 均可 |
| 内网穿透 | cpolar / ngrok 任一（推荐 cpolar，国内访问快） |

---

## 三、部署步骤

### 1. 克隆 MOSS-Image 代码库

```bash
git clone https://github.com/OpenMOSS/MOSS-Image.git
cd MOSS-Image
python -m venv .venv
# Windows: .venv\Scripts\activate
# Linux:   source .venv/bin/activate
pip install -r requirements.txt
pip install -e '.[ui]'
```

> ⚠️ Windows 下默认 `pip install torch` 装的是 **CPU 版**，需重装 CUDA 版：
> ```bash
> pip install --force-reinstall "torch==2.14.1+cu130" --index-url https://download.pytorch.org/whl/cu130
> ```

### 2. 下载模型权重

```bash
python -c "from huggingface_hub import snapshot_download; snapshot_download(repo_id='OpenMOSS-Team/MOSS-Image-Base', local_dir='models/MOSS-Image-Base')"
```

> 模型约 **25GB**，含 transformer / text_encoder / vae / tokenizer / scheduler。

### 3. 准备文生图 API 服务（FastAPI）

创建一个 `moss_image_api.py`，提供：
- `POST /generate`：接收提示词 JSON，返回 PNG。
- `GET /health`：健康检查。
- `X-API-Key` 鉴权。

核心逻辑（示意）：

```python
import torch
from fastapi import FastAPI, Header, HTTPException
from fastapi.responses import Response
from pydantic import BaseModel
import inference_runtime   # MOSS-Image 仓库自带加载器

MODEL = "models/MOSS-Image-Base"
app = FastAPI()

class Req(BaseModel):
    prompt: str
    negative_prompt: str = ""
    width: int = 1024
    height: int = 1024
    steps: int = 40
    guidance_scale: float = 3.0
    seed: int = -1

@app.post("/generate")
def generate(req: Req, x_api_key: str = Header(default="")):
    if x_api_key != "YOUR_API_KEY":
        raise HTTPException(401, "Invalid key")
    pipe = inference_runtime.load_pipeline(MODEL, local_files_only=True)
    image = pipe(req.prompt, width=req.width, height=req.height,
                 num_inference_steps=req.steps,
                 guidance_scale=req.guidance_scale,
                 generator=torch.Generator("cpu").manual_seed(req.seed)).images[0]
    buf = io.BytesIO(); image.save(buf, format="PNG")
    return Response(content=buf.getvalue(), media_type="image/png")
```

### 4. 按需启动守护（Guard）

创建一个守护进程，实现**按需拉起 + 闲置自动停**：

```python
# 伪代码
def on_request():
    if not main_service_alive():
        start_main_service()      # 拉起 moss_image_api.py
        wait_until_port_ready()   # 等 uvicorn 起来
    forward_request_to_main()     # 转发 /generate

def monitor():
    while True:
        if idle_timeout_exceeded():
            stop_main_service()   # 释放显存
        sleep(15)
```

关键点：
- **模型是懒加载**：只有实际调 `/generate` 才加载模型，所以守护只需等**端口就绪**即转发，不必等 `model_loaded`。
- 闲置计时从**请求进来**开始，且**转发中不判定闲置**，避免生成中途被停。

### 5. 配置内网穿透（以 cpolar 为例）

```bash
# 安装 cpolar 并配置 token（从 cpolar 后台获取）
cpolar authtoken <your-cpolar-authtoken>

# 配置隧道指向守护端口 (示例 8081)
# 编辑 ~/.cpolar/cpolar.yml
```

`cpolar.yml` 示意：

```yaml
authtoken: <your-cpolar-authtoken>
tunnels:
  website:
    proto: http
    addr: "8081"          # 指向守护端口
```

启动隧道：

```bash
cpolar start website -log stdout
```

日志中会输出公网地址：

```
Tunnel established at https://<your-tunnel-domain>.cpolar.top
```

---

## 四、云端 AI 调用方式

### 接口信息

| 项 | 值 |
|---|---|
| 接口地址 | `https://<your-tunnel-domain>.cpolar.top` |
| API Key | `<your-api-key>` |
| 生成接口 | `POST /generate` |
| 健康检查 | `GET /health` |
| 返回 | PNG 图片二进制 |

### curl

```bash
curl -X POST https://<your-tunnel-domain>.cpolar.top/generate \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-api-key>" \
  -d '{"prompt":"一只柴犬在草地上奔跑","width":512,"height":512,"steps":20}' \
  -o output.png
```

### Python

```python
import requests

r = requests.post(
    "https://<your-tunnel-domain>.cpolar.top/generate",
    headers={"Content-Type": "application/json", "X-API-Key": "<your-api-key>"},
    json={"prompt": "一只柴犬在草地上奔跑", "width": 512, "height": 512, "steps": 20},
    timeout=300,
)
if r.status_code == 200:
    open("output.png", "wb").write(r.content)
```

### 参数说明

| 参数 | 说明 | 默认 |
|---|---|---|
| `prompt` | 画面内容（支持中文） | 必填 |
| `negative_prompt` | 排除内容 | 空 |
| `width` / `height` | 尺寸，须为16的倍数 | 1024 |
| `steps` | 采样步数 1~100 | 40 |
| `guidance_scale` | 提示词遵循度 | 3.0 |
| `seed` | -1=随机，填数字可复现 | -1 |

---

## 五、注意事项

### 冷启动延迟
- 模型**按需加载**，冷启动首次调用约 **90~120 秒**。
- 之后连续调用快（模型驻留显存），闲置超时自动释放。

### 并发限制
- 显卡 12.9GB，一次处理 1 张，并行排队。
- 建议一次请求一张，多张换 `seed` 串行。

### 公网地址易变
- 免费内网穿透（cpolar 免费版）用**临时域名**，本机/隧道重启后地址会变。
- 需要**固定域名**可升级到 cpolar 基础版（保留子域名）。

### 鉴权与安全
- 必须携带 `X-API-Key`，否则返回 401。
- 公网地址勿公开分享，避免被滥用。

---

## 六、License

方案仅供参考与学习使用。模型权重遵循原项目（MOSS-Image / Apache 2.0）许可，请以官方仓库为准。