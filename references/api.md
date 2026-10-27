# Agnes AI API Reference

Current platform host: `https://api.agnes-ai.cn` (v1 paths below).
Legacy host `https://apihub.agnes-ai.com` remains a fallback for older keys.

Authentication: `Authorization: Bearer YOUR_API_KEY`
Content type: `application/json`

Full docs index: https://wiki.agnes-ai.cn/llms.txt

## Text — `agnes-3.0-flash`

Endpoint: `POST /v1/chat/completions`
(`POST /v1/responses` and Anthropic-compatible `POST /v1/messages` also exist; the script uses Chat Completions only.)

Required:

- `model`: fixed as `agnes-3.0-flash`
- `messages`: system/user/assistant messages; content may be plain text or blocks of `text` + `image_url` (image URL input supported)

Optional:

- `temperature`, `top_p`
- `stream`: boolean (SSE)
- `tools`, `tool_choice`: OpenAI-compatible function calling
- `chat_template_kwargs`: e.g. `{"enable_thinking": true}` — the script sends this by default (`--no-thinking` disables)

Notes:

- 512K context; up to 65,536 output tokens. The script does NOT send `max_tokens` (provider default budget applies).
- Thinking responses may include `choices[].message.reasoning_content` in addition to `content`.

Response: OpenAI-compatible; read `choices[].message.content`, `usage`.

## Image — `agnes-image-2.5-flash`

Endpoint: `POST /v1/images/generations`

Required:

- `model`: fixed as `agnes-image-2.5-flash`
- `prompt`
- `size`: tier `1K`/`2K`/`3K`/`4K` (script default `2K`) or legacy `WIDTHxHEIGHT` (unsupported exact sizes may be normalized by the provider, e.g. 1024x768 → 1152x864)

Optional:

- `ratio` (tier sizes only): `1:1 3:4 4:3 16:9 9:16 2:3 3:2 21:9` (script default `1:1`). Examples: 2K+1:1 = 2048x2048, 2K+16:9 = 2624x1472
- `extra_body.image`: array of input image URLs or Data URI Base64 for image-to-image / multi-image composition
- `extra_body.response_format`: `"url"` or `"b64_json"` — MUST stay inside `extra_body`, never top-level

Prompt structure:

`[Subject] + [Scene / Environment] + [Style] + [Lighting] + [Composition] + [Quality Requirements]`

For image-to-image, state what should change and what must remain unchanged.

For non-English user prompts, translate to English before sending the request. Preserve visual specifics and constraints.

Response: `data[0].url` or `data[0].b64_json`.

## Video — `agnes-video-2.5-flash`

Create task endpoint: `POST /v1/videos`

Recommended result endpoint: `GET /agnesapi?video_id={video_id}&model_name=agnes-video-2.5-flash` (works for all modes; a bare `video_id` query only works for `mode: "text"`)

Legacy task endpoint: `GET /v1/videos/{task_id}`

The video API is asynchronous. Create a task, then poll by `video_id` until `status` is `completed` or `failed`.

Required:

- `model`: fixed as `agnes-video-2.5-flash`
- `prompt`: text description of the video (use `<Picture N>` / `<Audio N>` placeholders for reference media)
- `mode`: `text` | `keyframe` | `reference`

Common optional fields:

- `seconds`: string `"4"`–`"12"` (default `"5"`)
- `size`: Flash only supports `"720P"`
- `aspect_ratio`: `21:9 16:9 4:3 1:1 3:4 9:16` (default `16:9`)
- `seed`: integer; `n`: only `1`

Mode-specific fields:

- `text`: no media fields allowed
- `keyframe`: `first_frame` and/or `last_frame` (image URLs, top-level); at least one required; `images`/`audios`/`videos` not allowed
- `reference`: `images` (max 5) and/or `audios` (max 3); `first_frame`/`last_frame`/`videos` not allowed (`videos` unsupported on Flash)

Mode/media mismatch or missing media returns 400.

Common status values:

- `queued`
- `in_progress`
- `completed`
- `failed`

The create response includes `id`/`task_id`, `video_id`, `progress` (0–100), `seconds`, and `size`. In live responses `video_id` may equal the `task_id` string; the script handles both lookup paths. The completed response carries the mp4 URL at top-level `url` (per docs) or `metadata.url` (observed live 2026-10-27); the script extracts `video_url`, `url`, `metadata.url`, and `remixed_from_video_id`.

Defaults used by the script: `seconds="5"`, `size="720P"`, `aspect_ratio="16:9"`, mode auto-inferred from provided media (frames → `keyframe`, images/audios → `reference`, else `text`). Reproducibility: set `seed`.

## Error Codes

- `400`: invalid request (bad params, mode/media mismatch, unsupported fields such as `width`/`height`/`fps`/`num_frames`/`num_inference_steps`/`quality`, pixel-size `size`, `aspect_ratio: "auto"`, `n != 1`)
- `401`: unauthorized; check API key
- `404`: task not found
- `500`: server error
- `503`: provider busy — commonly `{"code":"video_queue_full"}`; the script retries video create with backoff (6 attempts), otherwise retry later
