# MOSS-Image 图像生成 API — 调用方案

> 面向调用方（Bot / AI）。**纯方案文档**：所有地址、密钥、路径均为占位符，请替换为你的实际值。

---

## 一、接口信息（占位）

| 项目 | 值 |
|---|---|
| 接口地址（BASE_URL） | `https://<your-domain>.cpolar.cn` |
| 鉴权密钥（API Key） | `<your-api-key>` |
| 协议 | HTTP / JSON / 返回 PNG 或图片数组 |
| 地址查询 | `GET /info` 返回 `public_url` |

> ⚠️ 内网穿透（免费版）每次重启后公网地址会变。旧地址失效后，用查地址脚本或 `GET /info` 获取新地址。

---

## 二、生成图片（核心）

```
POST https://<your-domain>.cpolar.cn/generate
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

**返回**：单图返回 PNG 二进制；多图（batch）返回 JSON `{"images":[base64...], "job_id":N}`。

---

## 三、完整参数

| 参数 | 类型 | 说明 | 默认 |
|---|---|---|---|
| `prompt` | string | 画面内容 | 必填 |
| `negative_prompt` | string | 排除内容 | "" |
| `width`/`height` | int | 尺寸（16 倍数） | 768 |
| `steps` | int | 步数 1~100 | 15 |
| `guidance_scale` | float | 遵循度 | 3.0 |
| `seed` | int | -1=随机 | -1 |
| `sampler_name` | string | 采样器 | euler |
| `scheduler` | string | 调度器 | sgm_uniform |
| `loras` | array | 多 LoRA 叠加 | [] |
| `hires_fix` | bool | 两段式放大 | false |
| `hires_scale` | float | 放大倍数 | 1.5 |
| `batch` | array | 批量参数矩阵 | [] |
| `translate_to_en` | bool | 中英转译 | false |
| `init_image` | string | 图生图 base64 | 空 |
| `denoise` | float | 图生图强度 | 1.0 |
| `depth_map` | string | 深度控制图 base64 | 空 |
| `color_map` | string | 色彩/布局控制图 base64 | 空 |
| `control_strength` | float | 控制强度 0~10 | 1.0 |

---

## 四、高级功能

### 1. 多 LoRA 叠加
```json
{"loras": [{"name":"<lora_a>","weight":0.8},{"name":"<lora_b>","weight":0.4}]}
```

### 2. hires fix 两段式放大
```json
{"hires_fix": true, "hires_scale": 1.5}
```

### 3. 批量参数矩阵
```json
{"prompt":"未来城市","batch":[
  {"guidance_scale":3.0,"steps":10,"sampler_name":"euler"},
  {"guidance_scale":7.0,"steps":10,"sampler_name":"dpmpp_2m"}
]}
```

### 4. 深度 / 色彩控制
MOSS 自带控制，支持**深度控制**和**色彩/布局控制**（不支持姿势 openpose / 线稿 lineart，架构限制）。
```json
{
  "prompt":"a mountain landscape",
  "depth_map":"data:image/png;base64,....",
  "color_map":"data:image/png;base64,....",
  "control_strength":1.0
}
```

### 5. 图生图
```json
{"init_image":"data:image/png;base64,....","denoise":0.6}
```

---

## 五、查询接口

| 接口 | 说明 | 返回 |
|---|---|---|
| `GET /health` | 服务状态 | status/model/queue_len/samplers/loras |
| `GET /info` | **当前公网地址** | `{"public_url": "...", "model": "..."}` |
| `GET /queue` | 排队状态 | `{"queue_len":N,"status":"idle"/"busy"}` |
| `GET /history` | 历史记录 | count + 最近文件名 |

---

## 六、可用值

### 采样器（sampler_name）
```
euler, euler_ancestral, heun, dpm_2, dpm_2_ancestral, lms, dpm_fast, dpm_adaptive,
dpmpp_2s_ancestral, dpmpp_sde, dpmpp_sde_gpu, dpmpp_2m, dpmpp_2m_sde, dpmpp_2m_sde_gpu,
ddim, uni_pc, uni_pc_bh2
```

### 调度器（scheduler）
```
normal, karras, exponential, sgm_uniform, simple, ddim_uniform, beta, linear_quadratic, kl_optimal
```

### LoRA（lora_name）
- `<lora_color_depth>`：色彩/深度控制
- `<lora_edit>`：图像编辑

---

## 七、错误与可靠性

| 状态码 | 含义 | 处理 |
|---|---|---|
| 200 | 成功 | 响应体为 PNG / 图片数组 |
| 401 | API Key 错 | 检查 X-API-Key |
| 422 | 参数格式错 | 尺寸非16倍数、采样器名非法 |
| 502/504 | 生成失败/超时 | 已自动重试 3 次，仍失败才返回 |
| 404 | 地址失效 | **地址变了**，用查地址脚本或 /info 获取新地址 |

---

## 八、给调用方（AI）的说明

### 1. 地址自发现
- 服务能连上时，`GET /info` 返回当前公网地址和模型名。
- 若重启后地址变、AI 用旧地址连不上，则调不到 /info，需向部署方要新地址。

### 2. 出图速度
- 约 6-15 秒（视尺寸/步数/hires）。
- 单显卡 12.9GB，一次 1 张，多请求排队。

### 3. 冷启动
- 服务空闲时模型常驻显存；重启后首次出图约 30-60 秒（重新加载模型）。

### 4. 建议
- 一次只请求一张，多张用 `batch` 或换 `seed`。

---

## 九、部署方维护提示

- **架构**：Bot → 公网(内网穿透) → API服务 → ComfyUI(int8模型常驻) → 出图
- **地址落盘**：内网穿透日志写本地文件，`/info` 和查地址脚本读取
- **查地址脚本**：显示当前公网地址 + API Key
- **重启后取新地址**：运行查地址脚本，看 `Tunnel established at https://xxx`

---

> 请遵循模型原作者（MOSS-Image / Apache 2.0）许可。