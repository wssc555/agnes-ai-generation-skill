# Agnes AI Generation Skill

[中文](README.md)

Agnes AI Generation Skill is a standard Agent Skills package for calling Agnes AI text, image, and video generation APIs. It can be installed into Codex, Claude Code, OpenClaw, Cursor, Windsurf, and other clients that support Agent Skills. After installation, AI agents can use Agnes models for text generation, text-to-image, image-to-image, text-to-video, image-to-video, multi-image video, and keyframe animation workflows.

Official platform: [https://platform.agnes-ai.com/](https://platform.agnes-ai.com/)

Latest release: [v0.1.0 - Agnes Generation Skill](https://github.com/Yacey/agnes-ai-generation-skill/releases/tag/v0.1.0)

This release prioritizes the newer Agnes Video V2.0 `video_id` result lookup, returns generated media `urls` by default without downloading files, and documents that Agnes Responses API multi-turn function calling is not suitable for agent automatic tool loops yet. See [Releases](https://github.com/Yacey/agnes-ai-generation-skill/releases) for the full notes.

## Features

- Text generation with `agnes-3.0-flash` (Thinking mode ON by default, `--no-thinking` to disable)
- Streaming text responses
- OpenAI-compatible tool-calling request shape
- Text-to-image with `agnes-image-2.5-flash` (default `2K` tier + `1:1` ratio, `--size` tier/exact and `--ratio`)
- Image-to-image editing and multi-image composition with `agnes-image-2.5-flash`
- High-information-density image generation
- Text-to-video with `agnes-video-2.5-flash`
- Keyframe first/last-frame control (keyframe mode)
- Image-reference and audio-reference video (reference mode)
- Video duration control: `--seconds` (4–12, default 5)
- Auto mode inference: frames → keyframe, images/audios → reference, else text
- Prompt-based motion and scene control
- Cinematic video output
- Asynchronous video task creation and polling retrieval
- Seed-based reproducibility
- Automatic backoff retry on 503 video-queue-full
- Automatic English prompt translation for non-English image/video prompts

## Quick Start

### 1. Apply for an Agnes API Key

1. Open [https://platform.agnes-ai.com/](https://platform.agnes-ai.com/).
2. Register or sign in.
3. Create an API key from the platform.
4. Provide the API key to the AI in a trusted current session, or configure it as a local environment variable.

Do not commit API keys to Git, README files, screenshots, or public chat logs.

### 2. Install This Skill

Install into the current agent:

```powershell
npx skills add Yacey/agnes-ai-generation-skill
```

Install into all supported agents:

```powershell
npx skills add Yacey/agnes-ai-generation-skill --all
```

### 3. Configure the API Key

For the current PowerShell session:

```powershell
$env:AGNES_API_KEY="YOUR_API_KEY"
```

For persistent Windows user-level configuration:

```powershell
[Environment]::SetEnvironmentVariable("AGNES_API_KEY", "YOUR_API_KEY", "User")
```

The script also accepts:

- `AGNES_API_KEY`
- `AGNES_API_TOKEN`
- `APIHUB_AGNES_API_KEY`

### 4. Use the Skill

Ask your AI agent:

```text
Use Agnes to generate a high-information-density futuristic city image.
```

Or:

```text
Use Agnes to turn this image into a cinematic video.
```

## Usage Examples

Text:

```powershell
python scripts/agnes_api.py text --prompt "Write a concise product tagline for an AI assistant."
```

Text-to-image (defaults to `2K` tier + `1:1` ratio):

```powershell
python scripts/agnes_api.py image --prompt "A luminous floating city above a misty canyon at sunrise, cinematic realism"
```

Explicit tier size and ratio:

```powershell
python scripts/agnes_api.py image --prompt "A cinematic desktop wallpaper, clean lighting" --size 2K --ratio 16:9
```

`--size` accepts `1K`/`2K`/`3K`/`4K` tiers (default `2K`) or legacy exact sizes (e.g. `1024x768`; unsupported sizes may be normalized by the provider). `--ratio` applies to tier sizes only; supported: `1:1 3:4 4:3 16:9 9:16 2:3 3:2 21:9`.

Image-to-image:

```powershell
python scripts/agnes_api.py image --prompt "Turn the scene into a rainy cyberpunk night while preserving composition" --image https://example.com/input.png
```

Text-to-video (defaults: 5 seconds, 720P, 16:9):

```powershell
python scripts/agnes_api.py video --prompt "A cinematic shot of a cat walking on the beach at sunset" --poll
```

Custom duration and aspect ratio:

```powershell
python scripts/agnes_api.py video --prompt "Neon-lit street after rain, cinematic camera move" --seconds 8 --aspect-ratio 21:9 --poll
```

Image-reference video (mode auto-infers to `reference`; use `<Picture N>` placeholders in the prompt):

```powershell
python scripts/agnes_api.py video --prompt "Use <Picture 1> as the character reference, natural motion" --image https://example.com/character.png --poll
```

Audio-reference video (combinable with images, max 3 audios):

```powershell
python scripts/agnes_api.py video --prompt "Follow the rhythm of <Audio 1>, cinematic night driving" --audio https://example.com/music.mp3 --poll
```

Keyframe first/last-frame control (mode auto-infers to `keyframe`):

```powershell
python scripts/agnes_api.py video --prompt "Smooth transition from first frame to last frame" --first-frame https://example.com/first.png --last-frame https://example.com/last.png --poll
```

Retrieve a video task:

```powershell
python scripts/agnes_api.py video-get video_123456
```

Flash limits: `size` is fixed to `720P`, `seconds` is an integer 4–12, up to 5 reference images and 3 reference audios, reference videos unsupported. Create requests that hit `503 video_queue_full` are retried automatically with backoff (up to 6 attempts).

By default, command output is normalized with common fields such as `content`, `urls`, `translated_prompt`, and `next_steps`, while preserving the provider response under `raw`.

By default, return generated image or video `urls` directly. Do not download, save, or inspect media locally unless the user explicitly asks for a local file or visual inspection.

Streaming text output also includes aggregated `content`, event count, completion status, and a short raw prefix so the result is easier to inspect.

When the create response includes `video_id`, the script prefers the newer result endpoint: `/agnesapi?video_id=...&model_name=agnes-video-2.5-flash`. If `video_id` is a `task_...` string, the script falls back to the legacy `task_id` endpoint. For completed video tasks, the script extracts direct mp4 URLs from `video_url`, `url`, `metadata.url` (the new host was observed returning the mp4 there on 2026-10-27), or `remixed_from_video_id`, then places them in `urls`.

Note: Agnes Responses API multi-turn function calling is currently not suitable as the automatic tool-loop model for agents such as Codex or Claude Code. This skill's script uses the chat completions path; tool calling should be treated as request-shape compatibility rather than stable multi-turn tool execution.

Add `--raw` to print only the original Agnes response:

```powershell
--raw
```

Light smoke test. By default, this checks text, streaming text, tool-calling request shape, and text-to-image only. It does not create video tasks:

```powershell
python scripts/agnes_api.py smoke-test
```

The tool-calling request can be accepted by Agnes without returning `tool_calls` consistently. By default this is reported as a warning. Use strict mode to fail in that case:

```powershell
python scripts/agnes_api.py smoke-test --strict-tools
```

Image-to-image smoke test:

```powershell
python scripts/agnes_api.py smoke-test --include-image-edit
```

Single video case test:

```powershell
python scripts/agnes_api.py smoke-test --video-case text
```

Available video cases: `text`, `keyframe` (also runs image-to-image to obtain both frames), `reference-image`.

## Prompt Language

English prompts are more stable for Agnes video generation. For image and video calls, this skill automatically translates non-English prompts to English before sending them to the Agnes image/video APIs. It preserves subjects, scene details, style, lighting, composition, camera movement, motion, and constraints.

To disable automatic translation:

```powershell
python scripts/agnes_api.py video --prompt "non-English prompt" --no-translate-prompt
```

## Validation Status

After the 2026-10-27 migration to `https://api.agnes-ai.cn`, confirmed by live API:

- Basic text generation (`agnes-3.0-flash`)
- Streaming text
- Thinking mode (default on, and `--no-thinking`)
- Text-to-image default 2K+1:1 (measured 2048x2048)
- Explicit 2K+16:9 (measured 2624x1472)
- Legacy size 1024x768 normalized by the provider to 1152x864 (matches docs)
- Image-to-image
- Clear 400 error shape for invalid image URLs
- Non-English prompt auto-translation for text-to-image and text-to-video
- Completed text-to-video (720P, 4s) with mp4 URL extracted from `metadata.url`

Implemented per official docs with local validation-error tests, but not yet live-verified end-to-end (the provider returned sustained `503 video_queue_full` during testing; the free queue was saturated):

- Keyframe first/last-frame video
- Image-reference video
- Audio-reference video (never run live)

The queue-full error is transient provider congestion; the script retries with backoff. Test one mode at a time and keep provider errors visible.

## License

MIT License. See [LICENSE](LICENSE).
