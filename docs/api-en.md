# Senzhi AI developer guide · API, Feishu Base and ComfyUI

Use a catalog row to request a white-background packshot and a short listing description based on verified product facts. Each job returns a request ID, so the merchandising team can write the approved image and copy back to the correct SKU.

[Manage API keys in the workspace](https://www.senzhiai.com/workspace.php?tool=open_api) · [简体中文指南](api-zh.md) · [Product and batch generation guide](en.md)

## Three API calls from model selection to delivery

Use `https://www.senzhiai.com` as the base URL. Sign in to the workspace and create a key under API access. Send it from your server in the `Authorization: Bearer <key>` header or the `X-API-Key` header. The Bash and curl examples below read the key from the `SENZHI_API_KEY` environment variable.

| Step | Method and path | What to keep |
| --- | --- | --- |
| Discover models | `GET /api/open_models.php` | Model reference, supported parameters, pricing and example request |
| Create a job | `POST /api/open_generate.php` | The returned `request_id` |
| Retrieve a result | `GET /api/open_result.php?request_id=...` | An image or video file URL, or generated text |

### 1. Read the model catalog for your account

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  'https://www.senzhiai.com/api/open_models.php?type=image'
```

Set `type` to `image`, `text` or `video`, or omit it to list all available types. Reading the catalog is free. Choose an entry under `models`, check `pricing` and copy its `request_example`. The example uses a Senzhi `model_id` such as `db:123`. To select by display name, replace that field with `model_name` and use the exact spelling and capitalization. If the account has two models with the same name, the API returns `409 AMBIGUOUS_MODEL`. Select the intended model by ID.

### 2. Submit a white-background packshot

Save the JSON below as `request.json`. This example selects `nano-banana-pro` by name. Check that it appears in your account’s catalog, then replace the example image URL with your own. If you choose a different model, carry over its required options from `request_example`. Use a direct HTTPS image URL accessible to Senzhi. The order of `images` determines which reference your prompt calls image 1 or image 2.

```json
{
  "model_name": "nano-banana-pro",
  "prompt": "Create a front-facing white-background packshot of the cream cardigan in image 1. Preserve the collar, buttons and ribbed cuffs.",
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

One request creates one job and `n` is fixed at 1. Save the returned `request_id`. For example, `{"success":true,"request_id":"req_example","status":"queued"}` means the job is in the queue. Assign a new `idempotency_key` to each new job. If the connection drops before an ID arrives, repeat the **same request body with the same API key and idempotency key** to recover the accepted job.

### 3. Poll the original job and save its output

```bash
curl --silent --show-error --get \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  --data-urlencode 'request_id=req_example' \
  'https://www.senzhiai.com/api/open_result.php'
```

Replace `req_example` with the ID from the previous call and use the API key that created the job. Check the JSON `status` field on each response.

| Job status | Client action |
| --- | --- |
| `pending`, `queued`, `processing` | Poll again in 5–10 seconds. Follow `Retry-After` or `retry_after` when provided. |
| `success` | Download the original image or video from `result.url` promptly. Read generated text from `result.content`. |
| `failed` | Record `error.code` and `error.message`. Review the cause before creating another job. |

For a file download, set `RESULT_URL` to the full `result.url`, preserving its query string. For PNG output, run `curl --fail --location --proto '=https' --proto-redir '=https' --output knit-01.png "$RESULT_URL"`. Send only the file URL to the download host. The Senzhi API key belongs on requests to the Senzhi API domain.

After you have an ID, keep querying that job if polling times out or returns `503 MEDIA_UNAVAILABLE`. A `410 MEDIA_EXPIRED` response means the output has passed its retention period.

## Choosing parameters for images, text and video

On November 6, 2026, the standard and VIP account catalogs checked on the live site listed image and text models. The video workspace also offered MiniMax H3 and Agnes Video 2.5. The API supports video jobs once a model appears in your account’s `type=video` catalog. The video lists checked on that date were empty.

| Catalog task | Fields to check |
| --- | --- |
| Images · generate a hero image and a detail shot | Read `parameter_combinations` and `parameters`. Pick supported values for `options.resolution`, `aspect_ratio`, `quality` and `format`. For 4K, confirm the chosen combination offers that resolution. |
| Text · write multilingual listings from product facts | Put the language, length and verified facts in `prompt`. An optional `system_prompt` can set brand voice. Check image input support and the maximum image count before adding references. |
| Video · create a vertical clip for a product listing | Start with the video model’s `request_example`. Read `video_capabilities` for duration, resolution, aspect ratio and reference images. Keep `options.pricing_version`. Multiply the rate for the chosen resolution by the duration in seconds. If the price version changes, refresh the catalog and review the new quote. |

### Generate a product bullet for an overseas listing

Save this body as `request.json` and reuse the POST and polling commands above. Replace the idempotency key with one specific to this new job, and retain it if you retry the same submission. When the status is `success`, read the copy from `result.content`.

```json
{
  "model_name": "gpt-6.1-sol",
  "prompt": "Write one product bullet in English, up to 20 words. Verified details: this is a cream knit cardigan with four front buttons. Use only those facts.",
  "idempotency_key": "catalog-knit-01-copy-en-v1"
}
```

### Model display names verified on 6 November 2026

Copy the exact capitalization, spaces and brackets when using `model_name`. Query your own live catalog before submitting.

| Type | Names as shown in the catalog |
| --- | --- |
| Image | `nano-banana-pro` · `nano-banana-pro-vip（谷歌官方）` · `Nano Banana Pro【VIP】` |
| Image | `nano-banana-2` · `nano-banana-2.1` · `Nano-banana-2.1【VIP】` |
| Image | `GPT-image-2.5-sunburst【VIP】` · `GPT-image-2.5-flare【VIP】` · `GPT Image 2.0【VIP】` |
| Text | `gpt-6.1-sol` · `gemini-3.8-flash` · `gemini3.1-flash` |

## Feishu Base · start a job from a SKU row

Give a merchandiser a “Ready” status they can set when the brief and reference photo are approved. In Feishu Base, build the following workflow with the **Send HTTP request** automation action. [Feishu’s HTTP action guide](https://www.feishu.cn/hc/zh-CN/articles/410063847664)

1. **Set up fields.** Add SKU, brief version, reference image URL, prompt, model ID, status, request ID and output URL. If the photo requires a Feishu login, use an authorized service to obtain a direct link accessible to Senzhi, with enough validity for processing.
2. **Submit once.** Trigger the POST when a row enters “Ready” and its request ID is empty. Combine the record ID, task purpose and version into a stable idempotency key. Store the API key in restricted automation settings or on your own server. Parse the response as JSON, write the returned request ID to the row and set the status to “Processing.”
3. **Collect results separately.** Schedule a second workflow for unfinished rows with a request ID. Set the request URL to `https://www.senzhiai.com/api/open_result.php`. Add `request_id` in Feishu’s query parameter fields and map it to the row. On success, save `result.url` or `result.content`. On failure, save the error. For long-term storage, download originals into your team’s library and keep its permanent link in the row.

## ComfyUI · use Senzhi in a node workflow

The API workspace offers a **ComfyUI plug-in** and its installation guide. A catalog workflow can feed the output of a text node into an image node as a photographic brief, then preview or save the resulting image. [Get the plug-in and setup instructions](https://www.senzhiai.com/workspace.php?tool=open_api)

1. Extract the archive so that `ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/` contains `requirements.txt` directly.
2. Use the Python environment that runs ComfyUI to install the requirements. From the directory containing `ComfyUI/`, for example, run `python -m pip install -r ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/requirements.txt`.
3. Restart ComfyUI and refresh the browser. Search the canvas for `森智AI` or `NewAPI`. Configure the Senzhi service and your API key in the configuration node, choose an available model and connect the image, text or text preview nodes. Remove the key and inspect the exported JSON before sharing a workflow.
