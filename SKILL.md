---
name: agnes-ai-generation
description: Call Agnes AI / Sapiens AI generation APIs for text, image, and video. Use when the user asks to use Agnes models, Agnes 3.0 Flash, Agnes Image, Agnes Video, api.agnes-ai.cn, apihub.agnes-ai.com, or to generate text, images, edit images, create videos, animate images, create keyframe videos, or test Agnes API calls.
---

# Agnes AI Generation

Use this skill to call Agnes text, image, and video generation APIs. The current platform host is `https://api.agnes-ai.cn/v1`. The legacy host `https://apihub.agnes-ai.com` remains a fallback for older keys. Verified live on the new host (2026-10-27): text (3.0 Flash), image (2.5 Flash), and text-to-video with mp4 URL retrieval. Keyframe and reference video modes are implemented per docs but not yet live-verified.

## Quick Start

1. Read `references/api.md` when endpoint details, parameters, or response fields are needed.
2. Use `scripts/agnes_api.py` for real API calls instead of rewriting curl by hand.
3. Require an API key in `AGNES_API_KEY`, `AGNES_API_TOKEN`, or `APIHUB_AGNES_API_KEY`. Never print the key.
4. For light live verification, run `smoke-test`; it avoids video creation by default. Add `--include-image-edit` for image-to-image, and add `--video-case <case>` explicitly for video modes (`text`, `keyframe`, `reference-image`). Treat the skill as fully tested only when basic text, text streaming, text tool calling, text-to-image, image-to-image, and the needed video cases return successful responses.

## Commands

Text generation:

```bash
python scripts/agnes_api.py text --prompt "Write a concise product tagline for an AI assistant."
```

Streaming text:

```bash
python scripts/agnes_api.py text --prompt "Write a short product intro." --stream
```

Streaming output is normalized and includes aggregated `content`, `events`, `done`, and a short `raw_prefix`.

Text with Thinking mode explicitly disabled (Thinking is ON by default):

```bash
python scripts/agnes_api.py text --prompt "Summarize this file." --no-thinking
```

Thinking mode sends `chat_template_kwargs.enable_thinking=true`; responses may include a separate `reasoning_content` field in `raw.choices[0].message`. The `max_tokens` request field is omitted; the provider default output budget (up to 65,536 tokens) applies.

Image generation (defaults to 2K tier, 1:1 ratio):

```bash
python scripts/agnes_api.py image --prompt "A luminous floating city above a misty canyon at sunrise, cinematic realism"
```

Image generation with explicit tier size and ratio:

```bash
python scripts/agnes_api.py image --prompt "A cinematic desktop wallpaper, clean lighting, high detail" --size 2K --ratio 16:9
```

`--size` accepts tiers `1K`/`2K`/`3K`/`4K` (default `2K`) or legacy `WIDTHxHEIGHT` (e.g. `1024x768`). `--ratio` (default `1:1`) applies only to tier sizes; supported: `1:1 3:4 4:3 16:9 9:16 2:3 3:2 21:9`.

Image-to-image:

```bash
python scripts/agnes_api.py image --prompt "Turn the scene into a rainy cyberpunk night while preserving composition" --image https://example.com/input.png --size 1024x768
```

Text-to-video with polling (defaults: 5s, 720P, 16:9):

```bash
python scripts/agnes_api.py video --prompt "A cinematic shot of a cat walking on the beach at sunset" --poll
```

Custom duration and aspect ratio:

```bash
python scripts/agnes_api.py video --prompt "Neon-lit street after rain, cinematic camera move" --seconds 8 --aspect-ratio 21:9 --poll
```

Image reference (mode auto-infers to `reference`; use `<Picture N>` placeholders in the prompt):

```bash
python scripts/agnes_api.py video --prompt "Use <Picture 1> as the character reference, natural motion, keep appearance consistent" --image https://example.com/character.png --poll
```

Audio reference (combine with images freely, max 3 audios):

```bash
python scripts/agnes_api.py video --prompt "Follow the rhythm of <Audio 1>, cinematic night driving scene" --audio https://example.com/music.mp3 --poll
```

Keyframe first/last frame control (mode auto-infers to `keyframe`):

```bash
python scripts/agnes_api.py video --prompt "Smooth transition from the first frame pose to the last frame composition" --first-frame https://example.com/first.png --last-frame https://example.com/last.png --poll
```

Retrieve a video task (poll until `status` is `completed`; the mp4 URL may appear top-level or in `metadata.url` — observed live in `metadata.url`):

```bash
python scripts/agnes_api.py video-get video_123456
```

Light live smoke test:

```bash
python scripts/agnes_api.py smoke-test
```

Image edit smoke test:

```bash
python scripts/agnes_api.py smoke-test --include-image-edit
```

Single video smoke test:

```bash
python scripts/agnes_api.py smoke-test --video-case text
```

## Workflow

- Prefer `agnes-3.0-flash` for text chat/completions (`https://api.agnes-ai.cn/v1`). It supports 512K context, up to 65,536 output tokens, image URL input, tool calling, and Thinking mode (ON by default via `chat_template_kwargs.enable_thinking`; `--no-thinking` disables it; responses may include a separate `reasoning_content`). The `max_tokens` field is omitted; the provider output budget (up to 65,536 tokens) applies. Responses (`/v1/responses`) and Anthropic-compatible Messages (`/v1/messages`) endpoints exist but the script currently uses Chat Completions only.
- The previous text model `agnes-2.0-flash` on `apihub.agnes-ai.com` showed unreliable Responses API multi-turn function calling (provider could return `function_call` with overall `status=completed`, and `function_call_output` with `previous_response_id` could fail). Treat this caveat as unverified for `agnes-3.0-flash` until retested; prefer Chat Completions for text generation.
- Prefer `agnes-image-2.5-flash` for text-to-image, image-to-image, multi-image composition, and high-information-density image generation. Tiered sizes (`1K`/`2K`/`3K`/`4K`) combine with `ratio`; legacy exact sizes may be normalized by the provider. `response_format` must stay inside `extra_body`, never top-level. High-density generation is prompt-driven; include subject hierarchy, environment, secondary details, lighting, composition, and quality requirements.
- Prefer `agnes-video-2.5-flash` for text-to-video, keyframe first/last-frame control, and image/audio reference generation. `--mode` is auto-inferred from media (frames → `keyframe`, images/audios → `reference`, else `text`). Flash limits: `size` fixed to `720P`, `seconds` integer `"4"`–`"12"` (default `"5"`), `aspect_ratio` default `16:9` (`21:9 16:9 4:3 1:1 3:4 9:16`), reference images max 5, audios max 3, reference videos unsupported. Use `<Picture N>` / `<Audio N>` placeholders in the prompt for reference media.
- The provider may return `HTTP 503 video_queue_full` under load. The script retries create with backoff (6 attempts, up to ~10 min total); if it still fails, wait and retry later.
- For image and video generation, convert any non-English user prompt to a fluent English generation prompt before calling the image/video API. English prompts are more stable for Agnes video generation. Preserve concrete visual details, style, lighting, composition, motion, camera instructions, and constraints during translation.
- For videos, the API is asynchronous: create a task first, then poll or retrieve by `video_id`. Query with `model_name=agnes-video-2.5-flash` (`GET /agnesapi?video_id=...&model_name=...`) for all modes; a bare `video_id` query only works for `mode: "text"`. Only a `status` of `completed` makes the top-level `url` a deliverable video address.
- Warn the user before costly or long-running live video generation unless they explicitly asked to test or generate video.
- Test video capabilities one at a time with `smoke-test --video-case <case>` to avoid creating many tasks at once. Supported cases are `text`, `keyframe`, and `reference-image`.

## Current Validation Notes

- 2026-10-27: full migration to `https://api.agnes-ai.cn` — text `agnes-3.0-flash` (Thinking ON by default, `max_tokens` omitted), image `agnes-image-2.5-flash` (default `2K`/`1:1`, tier sizes + `--ratio`), video `agnes-video-2.5-flash` (4 modes, `--seconds`, `size` fixed 720P). Python syntax verified.
- Verified live on the new host: basic text, streaming text, Thinking mode (default on / `--no-thinking`), text-to-image (default 2K+1:1 = 2048x2048; 2K+16:9 = 2624x1472), legacy-size normalization (1024x768 → 1152x864), image-to-image, invalid-image-URL 400 shape, Chinese prompt auto-translation, completed text-to-video (4s, 720P) with mp4 URL extracted from `metadata.url`.
- Keyframe and image/audio reference video modes: implemented per docs, validation errors verified locally, but live end-to-end generation was blocked by sustained provider `503 video_queue_full`; pending user's own live testing.
- Audio-reference video mode has never been run live (no public audio asset was tested).
- Caveat: Agnes may accept tool-calling request parameters without consistently returning `tool_calls`; use `smoke-test --strict-tools` when strict tool-call validation is required.

## Output Handling

- Return generated image/video URLs directly by default. Do not download, save, open, or inspect generated media unless the user explicitly asks for a local file or visual inspection.
- For image responses, expect URL-style results when `extra_body.response_format` is `url`.
- For video responses, extract URLs from `video_url`, `url`, `metadata.url`, or `remixed_from_video_id` when `status` is `completed`. Live responses on the new host (2026-10-27) put the mp4 in `metadata.url`.
- For video retrieval, prefer `GET /agnesapi?video_id=...&model_name=agnes-video-2.5-flash`; legacy `GET /v1/videos/{task_id}` remains a fallback (and is what runs when the provider returns `task_...`-style `video_id`).
- If a request fails, report HTTP status and provider error body without exposing the API key.
