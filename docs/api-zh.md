# 森智 AI 开发指南｜API、飞书多维表格与 ComfyUI

把上新清单变成一组可追踪的任务。用一行 SKU 的参考图生成白底商品图，再用文字模型把已核实的商品资料写成英文卖点。两项任务各有独立的请求编号，成图和文案都能写回对应的商品记录。

[进入工作台管理 API Key](https://www.senzhiai.com/workspace.php?tool=open_api) · [English guide](api-en.md) · [产品与批量生图介绍](../README.md)

## 三个接口跑通一次生成

正式站地址为 `https://www.senzhiai.com`。登录工作台，从「API 接口调用」创建密钥。在自己的服务端通过 `Authorization: Bearer <密钥>` 鉴权，也可以用 `X-API-Key`。下面的终端命令使用 Bash 和 curl，从环境变量 `SENZHI_API_KEY` 读取密钥。

| 步骤 | 方法和路径 | 需要保存什么 |
| --- | --- | --- |
| 选模型 | `GET /api/open_models.php` | 模型标识、参数、报价和请求示例 |
| 建任务 | `POST /api/open_generate.php` | 本次任务返回的 `request_id` |
| 取结果 | `GET /api/open_result.php?request_id=...` | 图片、视频的原文件地址，或文字正文 |

### 1. 获取账号可用的模型

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  'https://www.senzhiai.com/api/open_models.php?type=image'
```

`type` 可选 `image`、`text` 或 `video`，省略会列出全部类型。读取目录免费。找到 `models` 数组中的目标模型，查看 `pricing` 并复制 `request_example`。示例中的 `model_id` 是本站编号，如 `db:123`。也可改用单独的 `model_name`，完整匹配目录的展示名称和大小写。如收到 `409 AMBIGUOUS_MODEL`，请改用目标模型的编号。

### 2. 提交一张商品白底图

把 JSON 保存为 `request.json`。下面用按名称调用的 `nano-banana-pro` 举例。先在账号目录确认该模型可用，将示例图片地址换成自己的地址。如果改用其他模型，请从它的 `request_example` 复制必填选项。参考图应是森智可以访问的 HTTPS 图片直链。`images` 的先后顺序对应提示词中的图 1、图 2。

```json
{
  "model_name": "nano-banana-pro",
  "prompt": "为图 1 的米色针织衫制作正面白底商品图，保留领口、纽扣和袖口纹理。",
  "images": ["https://example.com/products/knit-01.jpg"],
  "idempotency_key": "catalog-knit-01-white-v1"
}
```

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @request.json \
  'https://www.senzhiai.com/api/open_generate.php'
```

每次请求创建一个任务，参数 `n` 为 1。保存返回的 `request_id`。例如 `{"success":true,"request_id":"req_example","status":"queued"}` 中，`queued` 代表已入队。每个新任务使用新的 `idempotency_key`。若提交途中断线且未收到编号，用**原密钥、原标识和相同请求内容**重试，找回已受理的任务。

### 3. 查询原任务并保存结果

```bash
curl --silent --show-error --get \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  --data-urlencode 'request_id=req_example' \
  'https://www.senzhiai.com/api/open_result.php'
```

将 `req_example` 换成上一步的编号，使用提交时的同一把密钥。读取 JSON 中的 `status` 决定下一步。

| 任务状态 | 处理方式 |
| --- | --- |
| `pending`、`queued`、`processing` | 等待 5～10 秒再查。收到 `Retry-After` 或 `retry_after` 时按指定时间等待。 |
| `success` | 图片和视频读取 `result.url`，及时下载原文件。文字读取 `result.content`。 |
| `failed` | 保存 `error.code` 和 `error.message`，确认原因后决定是否发起新任务。 |

下载时，将响应里的完整 `result.url`（保留查询参数）赋给 `RESULT_URL`。例如，选择 PNG 输出时可运行 `curl --fail --location --proto '=https' --proto-redir '=https' --output knit-01.png "$RESULT_URL"`。临时文件链接自带访问凭证，下载时仅发送该链接，API Key 只发送到森智 API 域名。

拿到编号后，即使查询超时或返回 `503 MEDIA_UNAVAILABLE`，也继续查原任务。若返回 `410 MEDIA_EXPIRED`，表示该结果已过保存期限。

## 图片、文字和视频分别怎么填

2026 年 11 月 6 日核对时，正式站普通和 VIP 账号的 API 目录均列出图片和文字模型。网页视频工作台另有 MiniMax H3 和 Agnes Video 2.5。API 已支持视频任务协议，只有调用方的 `type=video` 目录列出模型后才可提交视频；本次核对的账号目录中视频列表为空。

| 用途 | 请求中需要关注的字段 |
| --- | --- |
| 图片 · 为同一件商品做主图和细节图 | 查看 `parameter_combinations` 和 `parameters`，按模型选 `options.resolution`、`aspect_ratio`、`quality`、`format`。需要 4K 时先确认所选组合提供该档位。 |
| 文字 · 从商品资料生成多语卖点 | 用 `prompt` 写目标语言、字数和商品事实，可加 `system_prompt` 约束语气。带图时先核对模型是否接受图片及张数上限。 |
| 视频 · 为商品详情页制作竖版短片 | 从视频模型的 `request_example` 开始，用 `video_capabilities` 核对时长、分辨率、比例及参考图。保留 `options.pricing_version`，按当前分辨率的每秒报价乘时长估算积分。价格版本变化时重新取目录并确认报价。 |

### 为独立站生成一条英文卖点

将下面的 JSON 另存为 `request.json`，使用与图片示例相同的 POST 和查询方法。把幂等标识换成本次新任务专用的值，同一次任务重试时保留它。状态为 `success` 后，从 `result.content` 读取英文正文。

```json
{
  "model_name": "gpt-6.1-sol",
  "prompt": "请用英语写一条不超过 20 个词的商品卖点。已核实的事实：米色针织开衫，前襟有四颗纽扣。只使用这些事实。",
  "idempotency_key": "catalog-knit-01-copy-en-v1"
}
```

### 可调用模型展示名（2026 年 11 月 6 日）

下面保留了目录里的大小写、空格和括号，可复制到 `model_name`。上线前请再查自己账号的实时目录。

| 类别 | 正式目录展示名 |
| --- | --- |
| 图片 | `nano-banana-pro` · `nano-banana-pro-vip（谷歌官方）` · `Nano Banana Pro【VIP】` |
| 图片 | `nano-banana-2` · `nano-banana-2.1` · `Nano-banana-2.1【VIP】` |
| 图片 | `GPT-image-2.5-sunburst【VIP】` · `GPT-image-2.5-flare【VIP】` · `GPT Image 2.0【VIP】` |
| 文字 | `gpt-6.1-sol` · `gemini-3.8-flash` · `gemini3.1-flash` |

## 飞书多维表格｜从上新清单发起任务

假设表里有一款针织衫，运营同事只需选中「待生成」即可交给流程处理。用飞书多维表格自动化的「发送 HTTP 请求」连接森智。以下是可自行搭建的字段和流程。[飞书 HTTP 请求说明](https://www.feishu.cn/hc/zh-CN/articles/410063847664)

1. **建字段。** 添加 SKU、版本、参考图直链、提示词、模型编号、状态、请求编号和结果链接。商品附件若需要登录飞书，先由有权限的服务取得森智能读取的临时直链，留足生成所需的有效期。
2. **提交并回写。** 当状态变为「待生成」且请求编号为空时，发送上面的 POST。用记录 ID、任务用途和版本拼成稳定的 `idempotency_key`。在受限的自动化配置或自己的服务端配置密钥，响应体选择 JSON，回写 `request_id`，状态改为「处理中」。
3. **单独查询。** 另一个定时流程只处理有编号的未完成行。请求 URL 填 `https://www.senzhiai.com/api/open_result.php`，在飞书的「查询参数」里把 `request_id` 映射到该行编号。成功后回写 `result.url` 或 `result.content`，失败后记录原因。要长期留档，可把下载的原文件存入团队素材库，再把归档地址写回该行。

## ComfyUI｜把商品图接入节点工作流

森智工作台「API 接口调用」页面提供 **ComfyUI 插件** 下载和安装指引。已有商品图工作流时，可以把文字节点输出的拍摄要求送给图片节点，再预览或保存生成结果。[获取插件与安装说明](https://www.senzhiai.com/workspace.php?tool=open_api)

1. 解压插件，让插件目录位于 `ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/`，目录内可见 `requirements.txt`。
2. 在运行 ComfyUI 的 Python 环境里安装依赖。例如，在包含 `ComfyUI/` 的目录执行 `python -m pip install -r ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/requirements.txt`。
3. 重启 ComfyUI 并刷新浏览器，在画布上搜索「森智AI」或「NewAPI」。用配置节点填写森智服务和 API Key，选择账号可用的模型，再连接图片模型接口、文本模型接口或文本预览节点。分享工作流前清除密钥并检查导出的 JSON。
