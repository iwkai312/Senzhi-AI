# Senzhi AI 개발 가이드 | API, 페이수 다차원 표, ComfyUI

상품 관리표에 등록한 SKU와 사진을 바탕으로 대표 이미지를 만들고 결과를 같은 행에 기록할 수 있습니다. 아래는 이미지 모델을 조회하고 흰 배경 상품 컷 한 장을 요청한 뒤 원본을 받는 과정입니다.

[API 키 관리](https://www.senzhiai.com/workspace.php?tool=open_api) · [제품 및 일괄 생성 가이드](ko.md) · [English](api-en.md)

## 1. 계정의 모델 목록 조회

작업 공간에 로그인해 API 메뉴에서 키를 만드세요. 아래 Bash와 curl 예제는 서버의 `SENZHI_API_KEY` 환경 변수에서 키를 읽습니다. 인증할 때는 `Authorization: Bearer <key>` 또는 `X-API-Key` 헤더를 사용합니다.

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  'https://www.senzhiai.com/api/open_models.php?type=image'
```

`type`은 `image`, `text`, `video` 중 하나이며 생략하면 전체 목록을 조회합니다. 목록 조회는 무료입니다. `models` 배열에서 원하는 모델의 `pricing`과 `request_example`을 확인하세요. `model_id`에는 `db:123`처럼 Senzhi에서 발급한 식별자를 사용합니다. 이름으로 호출할 때는 `model_id` 대신 대소문자까지 정확히 일치하는 `model_name`을 사용하세요. 이름이 겹쳐 `409 AMBIGUOUS_MODEL`이 반환되면 모델 ID를 지정합니다.

## 2. 흰 배경 상품 컷 한 장 요청

아래 JSON을 `request.json`으로 저장하고 예시 이미지 URL을 자신의 상품 사진 주소로 바꾸세요. Senzhi가 접근할 수 있는 HTTPS 이미지 직접 링크를 사용합니다. `images` 배열 순서가 프롬프트의 ‘이미지 1’, ‘이미지 2’에 해당합니다. 모델을 바꿀 때는 그 모델의 `request_example`에 있는 필수 옵션을 함께 사용하세요.

```json
{
  "model_name": "nano-banana-pro",
  "prompt": "이미지 1에 있는 크림색 카디건의 정면 흰 배경 상품 사진을 만들어 주세요. 옷깃, 단추, 소매 끝의 골지 짜임을 살려 주세요.",
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

요청 하나가 작업 한 건을 만들며 `n`은 1입니다. 예시 응답 `{"success":true,"request_id":"req_example","status":"queued"}`에서 `request_id`를 저장하세요. 새 작업에는 새로운 `idempotency_key`를 부여합니다. 작업 ID를 받기 전에 연결이 끊기면 **같은 API 키, 같은 멱등 키, 같은 요청 본문**으로 다시 보내 기존에 접수된 작업을 찾습니다.

## 3. 원래 작업 조회 및 원본 저장

`req_example`을 직전에 받은 ID로 바꾸고 작업을 제출한 API 키로 조회합니다.

```bash
curl --silent --show-error --get \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  --data-urlencode 'request_id=req_example' \
  'https://www.senzhiai.com/api/open_result.php'
```

| JSON의 `status` | 처리 방법 |
| --- | --- |
| `pending` / `queued` / `processing` | 5~10초 뒤 다시 조회합니다. `Retry-After` 또는 `retry_after`가 있으면 해당 대기 시간을 따릅니다. |
| `success` | 이미지·영상은 `result.url`, 텍스트는 `result.content`를 읽습니다. 파일 URL의 쿼리 문자열을 보존해 원본을 내려받으세요. |
| `failed` | `error.code`와 `error.message`를 저장하고 원인을 확인합니다. |

조회가 시간 초과되거나 `503 MEDIA_UNAVAILABLE`가 반환되면 기존 ID로 계속 조회하세요. `410 MEDIA_EXPIRED`는 결과 보관 기간이 끝났다는 의미입니다. 파일을 내려받을 때는 `result.url`만 사용하고, Senzhi API 키는 Senzhi API 도메인에만 전송하세요.

## 이미지·텍스트·영상별 입력

- **이미지**는 `parameter_combinations`와 `parameters`에서 해상도, 비율, 품질, 포맷의 가능한 조합을 확인합니다. 4K가 필요하면 모델이 해당 조합을 제공하는지 확인하세요.
- **텍스트**는 `prompt`에 검증된 제품 정보, 작성 언어, 분량을 넣습니다. `system_prompt`로 브랜드 문체를 지정할 수 있습니다. 참고 이미지는 모델의 입력 지원 여부와 허용 장수를 확인한 뒤 추가하세요.
- **영상**은 계정의 `type=video` 목록에 모델이 표시될 때 호출할 수 있습니다. 2026년 11월 6일 확인한 API 목록에는 영상 모델이 없었으며, MiniMax H3와 Agnes Video 2.5는 웹 작업 공간에서 제공됐습니다. API 영상 모델을 사용할 때는 `video_capabilities`에서 길이, 해상도, 참고 이미지를 확인하고 예제의 `options.pricing_version`을 유지하세요. 선택한 해상도의 초당 가격과 길이로 비용을 계산합니다.

### 해외 상품 페이지에 넣을 영문 문구 작성

다음 본문을 `request.json`으로 저장하고 이미지 예제의 POST와 조회 명령을 그대로 사용하세요. 새 작업마다 전용 `idempotency_key`로 바꾸고, 같은 요청을 재전송할 때는 원래 값을 유지하세요. 상태가 `success`가 되면 `result.content`에서 영문 문구를 읽습니다.

```json
{
  "model_name": "gpt-6.1-sol",
  "prompt": "영어로 상품 소개 문장 하나를 20단어 이내로 작성해 주세요. 확인된 사실은 크림색 니트 카디건이며 앞면에 단추가 네 개 있다는 것입니다. 이 사실만 사용하세요.",
  "idempotency_key": "catalog-knit-01-copy-en-v1"
}
```

### API 모델 표시 이름 (2026년 11월 6일 확인)

`model_name`에는 대소문자, 띄어쓰기, 괄호를 포함하여 표시 이름을 그대로 입력하세요. 제출 전에는 내 계정의 최신 목록을 확인하세요.

| 유형 | 목록의 표시 이름 |
| --- | --- |
| 이미지 | `nano-banana-pro` · `nano-banana-pro-vip（谷歌官方）` · `Nano Banana Pro【VIP】` |
| 이미지 | `nano-banana-2` · `nano-banana-2.1` · `Nano-banana-2.1【VIP】` |
| 이미지 | `GPT-image-2.5-sunburst【VIP】` · `GPT-image-2.5-flare【VIP】` · `GPT Image 2.0【VIP】` |
| 텍스트 | `gpt-6.1-sol` · `gemini-3.8-flash` · `gemini3.1-flash` |

## 페이수 다차원 표(Feishu Base) 자동화

1. SKU, 참고 이미지 URL, 지시문, 버전, 모델, 상태, 작업 ID, 결과 열을 만드세요. 첨부 파일에 페이수 로그인이 필요하면 권한 있는 서비스에서 가져온 직접 링크를 쓰고 생성 시간보다 넉넉한 유효 기간을 확보합니다.
2. ‘제작 승인’이고 작업 ID가 빈 행만 자동화 대상으로 설정하세요. ‘发送 HTTP 请求’(HTTP 요청 보내기)로 위 POST를 보내고, 행 ID·작업 용도·버전으로 고정된 `idempotency_key`를 만듭니다. `request_id`를 해당 행에 기록합니다. API 키는 권한이 제한된 자동화나 자체 서버에서 관리하세요.
3. 별도 예약 작업에서 ID가 있는 미완료 행을 조회합니다. GET의 `request_id` 쿼리 항목에 기록한 ID를 전달하고 성공 시 `result.url`이나 `result.content`를 같은 행에 적습니다. 장기 보관할 파일은 팀의 자료실로 복사하세요.

## ComfyUI에서 노드 연결

작업 공간의 API 메뉴에서 플러그인을 받은 뒤 `ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/` 바로 아래에 `requirements.txt`가 있도록 압축을 풉니다. ComfyUI가 사용하는 Python 환경에서 해당 패키지를 설치하고 ComfyUI를 재시작한 뒤 브라우저를 새로 고치세요. ‘森智AI’나 ‘NewAPI’를 검색해 설정 노드에 접속 정보를 입력하고 이미지, 텍스트, 텍스트 미리보기 노드를 연결합니다. 워크플로를 공유하기 전 키를 지우고 내보낸 JSON도 확인하세요. [플러그인 및 설치 안내](https://www.senzhiai.com/workspace.php?tool=open_api)
