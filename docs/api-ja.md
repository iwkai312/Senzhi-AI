# 森智 AI 開発ガイド｜API・飛書多維表格・ComfyUI

商品管理表の1行から白背景の画像を作り、完成したファイルを同じSKUに登録する手順です。画像やテキストを書き戻すには、モデルの取得、生成の依頼、結果の照会の3つの操作を組み合わせます。

[APIキーの管理](https://www.senzhiai.com/workspace.php?tool=open_api) · [製品と一括生成のガイド](ja.md) · [English](api-en.md)

## 1. このアカウントで使えるモデルを調べる

ワークスペースにログインし、API画面でキーを作成します。次の例はBashとcurlを使い、サーバーに設定した環境変数 `SENZHI_API_KEY` からキーを読み取ります。リクエストの認証には `Authorization: Bearer <key>` または `X-API-Key` ヘッダーを使用します。

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  'https://www.senzhiai.com/api/open_models.php?type=image'
```

`type` には `image`、`text`、`video` を指定でき、省略すると全件を取得します。モデル一覧の照会は無料です。`models` 配列から目的のモデルを探し、`pricing` と `request_example` を確認してください。`model_id` には `db:123` のような森智の識別子を指定します。表示名で選ぶ場合は代わりに `model_name` を使い、大文字・小文字も一覧とそろえます。同名のモデルで `409 AMBIGUOUS_MODEL` が返った場合は識別子を指定してください。

## 2. 商品画像を1枚依頼する

次のJSONを `request.json` として保存し、サンプルのURLを商品画像のURLに置き換えます。画像は森智 AI が読み取れるHTTPSの直リンクを用意してください。`images` の順番が指示文の「画像1」「画像2」に対応します。モデルを変更する際は、取得した `request_example` に合わせて必須項目も調整します。

```json
{
  "model_name": "nano-banana-pro",
  "prompt": "画像1の生成り色のカーディガンを正面から撮った白背景の商品画像を作成してください。襟、ボタン、袖口のリブを保ってください。",
  "images": [
    "https://example.com/products/knit-01.jpg"
  ],
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

1件の依頼に対して `n` は1です。応答例 `{"success":true,"request_id":"req_example","status":"queued"}` の `request_id` をSKUにひも付けて保存します。新しい依頼には新しい `idempotency_key` を設定してください。応答が届く前に通信が切れた場合は、同じAPIキー、同じ `idempotency_key`、同じリクエスト本文で再送し、受理済みの依頼を取得します。

## 3. 元の依頼を照会して原本を保存する

`req_example` を受け取ったIDに置き換え、送信時のAPIキーで照会します。

```bash
curl --silent --show-error --get \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  --data-urlencode 'request_id=req_example' \
  'https://www.senzhiai.com/api/open_result.php'
```

| JSONの `status` | 次の操作 |
| --- | --- |
| `pending` / `queued` / `processing` | 5～10秒後に再照会します。`Retry-After` または `retry_after` を受け取ったら、その待ち時間に従います。 |
| `success` | 画像・動画は `result.url`、テキストは `result.content` を取得します。署名付きURLはクエリ文字列を含めてそのまま使い、原本を早めに保存します。 |
| `failed` | `error.code` と `error.message` を記録し、原因を確認します。 |

照会のタイムアウトや `503 MEDIA_UNAVAILABLE` が発生したら、受け取った元のIDで照会を続けます。`410 MEDIA_EXPIRED` は保存期間が過ぎたことを示します。ファイルのダウンロード先には `result.url` だけを渡し、森智のAPIキーは森智のAPIドメインへのリクエストにだけ付けます。

## 画像・テキスト・動画の設定

- **画像**は `parameter_combinations` と `parameters` を調べ、希望する解像度・縦横比・品質・形式を組み合わせます。4Kは対応する設定で選べます。
- **テキスト**は `prompt` に商品の事実、対象言語、文字数を書きます。文体は `system_prompt` で指定できます。参考画像を添える場合は、そのモデルの対応と枚数を確認してください。
- **動画**はアカウントの `type=video` にモデルが掲載されたときに使えます。2026年11月6日に確認したAPI一覧の動画欄は空で、MiniMax H3とAgnes Video 2.5はWeb版で利用できました。APIから使う際は `video_capabilities` で尺・画質・参照画像を確認し、例にある `options.pricing_version` を保持します。料金は選んだ解像度の毎秒単価と秒数で決まります。

### 海外向け商品ページの特長を英語で書く

次の本文を `request.json` に保存し、商品画像の例と同じPOSTと照会コマンドを使います。新しい依頼には専用の `idempotency_key` を設定し、同じ依頼を再送するときは元の値を使ってください。状態が `success` になったら、`result.content` から英文を取得します。

```json
{
  "model_name": "gpt-6.1-sol",
  "prompt": "商品ページに載せる英語の説明を20語以内で1文書いてください。確認済みの事実は、クリーム色のニットカーディガンで、前にボタンが4つあることです。この情報のみを使ってください。",
  "idempotency_key": "catalog-knit-01-copy-en-v1"
}
```

### APIモデル名一覧（2026年11月6日確認）

`model_name` には大文字・小文字、空白、括弧を含めて下記の表記をそのまま使います。送信前に自分のアカウントの最新一覧を確認してください。

| 種類 | 一覧の表示名 |
| --- | --- |
| 画像 | `nano-banana-pro` · `nano-banana-pro-vip（谷歌官方）` · `Nano Banana Pro【VIP】` |
| 画像 | `nano-banana-2` · `nano-banana-2.1` · `Nano-banana-2.1【VIP】` |
| 画像 | `GPT-image-2.5-sunburst【VIP】` · `GPT-image-2.5-flare【VIP】` · `GPT Image 2.0【VIP】` |
| テキスト | `gpt-6.1-sol` · `gemini-3.8-flash` · `gemini3.1-flash` |

## 飛書多維表格（Feishu Base）へ結果を書き戻す

1. SKU、制作指示、画像URL、版番号、モデル、状態、タスクID、結果の列を作ります。飛書へのログインを求める添付URLは、権限を持つサービスから取得した、生成に必要な有効期間の直リンクに置き換えてください。
2. 承認済みでタスクIDが空の行を対象に、「发送 HTTP 请求」（HTTPリクエストの送信）から上記のPOSTを送ります。行ID・用途・版番号を組み合わせた固定の `idempotency_key` を使い、応答の `request_id` を同じ行に保存します。APIキーはアクセスを制限した自動化の設定か、自社のサーバー側に保管します。
3. 別の定期処理で未完了の行を抽出し、GETの `request_id` パラメーターに行内のIDを設定します。完了時に `result.url` か `result.content` を書き戻します。長期保管には原本を素材庫にコピーします。

## ComfyUIで商品制作のノードを組む

ワークスペースのAPI画面からプラグインをダウンロードし、`ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/` の直下に `requirements.txt` がある形で展開します。ComfyUIを実行するPython環境で依存パッケージをインストールし、ComfyUIを再起動してブラウザーを更新します。「森智AI」または「NewAPI」を検索し、設定ノードに接続情報を入力すると、画像・テキスト・テキストプレビューのノードを使えます。共有前にはAPIキーを削除し、書き出したJSONも確認してください。[プラグインとインストール手順](https://www.senzhiai.com/workspace.php?tool=open_api)
