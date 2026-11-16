# MOSS-Image API — 调用用法示意

> 这是一份**纯示意**文档：演示如何通过本地 MOSS-Image 服务生成图片。
> 所有地址、密钥、文件名均为**占位符**，请替换为你自己的实际值。

---

## 一、基础信息（示意）

| 项 | 示意值 | 说明 |
|---|---|---|
| 接口地址 | `https://<your-domain>.cpolar.top` | 你的公网隧道地址 |
| API Key | `<your-api-key>` | 鉴权密钥 |

> ⚠️ 实际使用时把 `<your-domain>`、`<your-api-key>` 换成你部署时的真实值。

---

## 二、文生图（Text to Image）

生成全新图片。

```
POST https://<your-domain>.cpolar.top/generate
Headers:
  Content-Type: application/json
  X-API-Key: <your-api-key>
Body:
{
  "prompt": "一只柴犬在草地上奔跑，阳光明媚",
  "negative_prompt": "",
  "width": 768,
  "height": 768,
  "steps": 15,
  "guidance_scale": 3.0,
  "seed": -1
}
```

**返回**：PNG 图片二进制（HTTP 200）。

### curl 示例
```bash
curl -X POST https://<your-domain>.cpolar.top/generate \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-api-key>" \
  -d '{"prompt":"一只柴犬在草地上奔跑","width":768,"height":768,"steps":15}' \
  -o output.png
```

---

## 三、图生图（Image to Image / img2img）

基于一张已有图片重绘/编辑。需要把原图转成 **base64 data URL** 传入 `init_image`。

```
POST https://<your-domain>.cpolar.top/generate
Headers:
  Content-Type: application/json
  X-API-Key: <your-api-key>
Body:
{
  "prompt": "改成夜晚霓虹风格",
  "init_image": "data:image/png;base64,<原图base64编码>",
  "denoise": 0.6,
  "width": 512,
  "height": 512,
  "steps": 15
}
```

- `init_image`：原图的 base64 data URL（`data:image/png;base64,....`）
- `denoise`：重绘强度，`1.0`=完全重绘，`0.3~0.6`=保留原图结构做微调

### Python 示例
```python
import requests, base64

with open("input.png", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()
data_url = "data:image/png;base64," + b64

r = requests.post(
    "https://<your-domain>.cpolar.top/generate",
    headers={"Content-Type": "application/json", "X-API-Key": "<your-api-key>"},
    json={"prompt": "改成夜晚霓虹风格", "init_image": data_url, "denoise": 0.6,
          "width": 512, "height": 512, "steps": 15},
    timeout=120,
)
open("output.png", "wb").write(r.content)
```

---

## 四、采样器 / 调度器 控制

可切换不同采样器与调度器，影响出图风格和速度。

```
POST https://<your-domain>.cpolar.top/generate
Headers:
  X-API-Key: <your-api-key>
Body:
{
  "prompt": "未来城市",
  "sampler_name": "dpmpp_2m",
  "scheduler": "karras",
  "steps": 15
}
```

### 可用采样器（示意，实际以服务返回为准）
```
euler, euler_ancestral, heun, dpm_2, dpmpp_2m, dpmpp_2m_sde,
dpmpp_2s_ancestral, dpmpp_sde, ddim, uni_pc, uni_pc_bh2 ...
```

### 可用调度器（示意）
```
normal, karras, sgm_uniform, simple, ddim_uniform, beta ...
```

---

## 五、LoRA 支持

可加载 LoRA 适配器增强效果（色彩/深度控制、图像编辑等）。

```
POST https://<your-domain>.cpolar.top/generate
Headers:
  X-API-Key: <your-api-key>
Body:
{
  "prompt": "雪山湖泊",
  "lora_name": "moss_image_color_depth_control.safetensors",
  "lora_strength": 1.0
}
```

- `lora_name`：LoRA 文件名（可留空 = 不使用）
- `lora_strength`：LoRA 强度（0~1）

---

## 六、完整参数一览

| 参数 | 类型 | 说明 | 默认 |
|---|---|---|---|
| `prompt` | string | 画面内容（支持中文） | 必填 |
| `negative_prompt` | string | 排除内容 | "" |
| `width` | int | 宽度（16 的倍数） | 768 |
| `height` | int | 高度（16 的倍数） | 768 |
| `steps` | int | 采样步数 1~100 | 15 |
| `guidance_scale` | float | 提示词遵循度 | 3.0 |
| `seed` | int | -1=随机，填数字可复现 | -1 |
| `sampler_name` | string | 采样器 | euler |
| `scheduler` | string | 调度器 | sgm_uniform |
| `lora_name` | string | LoRA 文件名（可空） | "" |
| `lora_strength` | float | LoRA 强度 | 1.0 |
| `init_image` | string | 图生图：原图 base64 | 空（文生图） |
| `denoise` | float | 图生图去噪强度 0~1 | 1.0 |

---

## 七、错误对照

| 状态码 | 含义 | 处理 |
|---|---|---|
| 200 | 成功 | 响应体为 PNG |
| 401 | Key 错误 | 检查 X-API-Key |
| 422 | 参数错误 | 检查尺寸是否 16 倍数、采样器名是否合法 |
| 502/504 | 服务端失败/超时 | 重试 |

---

## 说明

- 本文件仅示意**调用用法**，不包含任何真实部署环境、地址、密钥或模型文件。
- 具体可用值（采样器/调度器/LoRA 列表）以你的服务 `/health` 接口返回为准。
- 请遵循模型原作者（MOSS-Image / Apache 2.0）许可。