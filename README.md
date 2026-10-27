# Agnes AI Generation Skill

[English](README_EN.md)

用于 Agent Skills 的 Agnes AI 生成技能，封装 Agnes 官方文本、图片、视频 API。它采用标准 `SKILL.md` 结构，可安装到 Codex、Claude Code、OpenClaw、Cursor、Windsurf 等支持 Agent Skills 的客户端。安装后，你可以让 AI 直接调用 Agnes 模型完成文生图、图生图、文生视频、图生视频、多图视频、关键帧动画等操作。

官网与 API 平台：[https://platform.agnes-ai.com/](https://platform.agnes-ai.com/)

最新版本：[v0.1.0 - Agnes Generation Skill](https://github.com/Yacey/agnes-ai-generation-skill/releases/tag/v0.1.0)

本版本重点修复：Agnes Video V2.0 优先使用新版 `video_id` 查询接口；生成图片/视频后默认只返回 `urls`，不主动下载媒体；明确 Agnes Responses API 多轮函数调用暂不适合作为 agent 自动工具循环。完整说明见 [Releases](https://github.com/Yacey/agnes-ai-generation-skill/releases)。

## 功能

- 文本生成：`agnes-3.0-flash`（Thinking 模式默认开启，可用 `--no-thinking` 关闭）
- 流式文本响应
- OpenAI 兼容的工具调用请求结构
- 文生图：`agnes-image-2.5-flash`（默认 `2K` 档位 + `1:1` 画幅，支持 `--size` 档位/精确尺寸与 `--ratio`）
- 图生图 / 图片编辑 / 多图合成：`agnes-image-2.5-flash`
- 高信息密度图片生成
- 文生视频：`agnes-video-2.5-flash`
- 首尾帧控制视频（keyframe 模式）
- 图片参考 / 音频参考视频（reference 模式）
- 视频时长控制：`--seconds`（4–12 秒，默认 5）
- 模式自动推断：提供帧图 → keyframe，提供图/音参考 → reference，否则 text
- 基于提示词的运动与场景控制
- 电影感视觉输出
- 异步视频任务创建与轮询检索
- 基于 seed 的可重复生成
- 503 视频队列满自动退避重试
- 自动将非英文图片/视频提示词翻译为英文提示词，提高 Agnes 视频生成稳定性

## 快速开始

### 1. 申请 Agnes API Key

1. 打开 Agnes API 平台：[https://platform.agnes-ai.com/](https://platform.agnes-ai.com/)
2. 注册或登录账号。
3. 在平台中申请 / 创建 API Key。
4. 拿到 API Key 后，可以在可信任的当前 AI 会话里发送给 AI，或配置为本机环境变量。

安全提醒：不要把 API Key 写入 Git 仓库、README、截图或公开聊天记录。

### 2. 安装本 Skill

安装到当前 Agent：

```powershell
npx skills add Yacey/agnes-ai-generation-skill
```

安装到所有支持的 Agent：

```powershell
npx skills add Yacey/agnes-ai-generation-skill --all
```

安装后，当你要求已安装的 AI Agent 使用 Agnes 生成图片或视频时，AI 会自动触发本 skill。

### 3. 配置 API Key

临时配置当前 PowerShell 会话：

```powershell
$env:AGNES_API_KEY="YOUR_API_KEY"
```

Windows 用户级持久配置：

```powershell
[Environment]::SetEnvironmentVariable("AGNES_API_KEY", "YOUR_API_KEY", "User")
```

脚本也会识别以下变量名：

- `AGNES_API_KEY`
- `AGNES_API_TOKEN`
- `APIHUB_AGNES_API_KEY`

### 4. 开始使用

你可以直接对 AI 说：

```text
使用 Agnes 帮我生成一张高信息密度的未来城市图片。
```

或：

```text
使用 Agnes 把这张图片生成一段电影感视频。
```

如果你已经把 API Key 发送给当前 AI，AI 可以配置环境变量并调用本 skill。之后即可解锁生图、生视频等相关操作。

## 命令示例

文本生成：

```powershell
python scripts/agnes_api.py text --prompt "Write a concise product tagline for an AI assistant."
```

文生图（默认 `2K` 档位 + `1:1` 画幅）：

```powershell
python scripts/agnes_api.py image --prompt "A luminous floating city above a misty canyon at sunrise, cinematic realism"
```

指定档位尺寸与画幅比例：

```powershell
python scripts/agnes_api.py image --prompt "A cinematic desktop wallpaper, clean lighting" --size 2K --ratio 16:9
```

`--size` 支持 `1K`/`2K`/`3K`/`4K` 档位（默认 `2K`）或历史精确尺寸（如 `1024x768`，不支持的精确尺寸可能被服务端标准化）；`--ratio` 仅用于档位尺寸，支持 `1:1 3:4 4:3 16:9 9:16 2:3 3:2 21:9`。

中文提示词文生图，脚本会先自动翻译成英文：

```powershell
python scripts/agnes_api.py image --prompt "一座高信息密度的未来城市集市，拥挤人群，飞行汽车，全息招牌，电影感写实风格"
```

图生图：

```powershell
python scripts/agnes_api.py image --prompt "Turn the scene into a rainy cyberpunk night while preserving composition" --image https://example.com/input.png
```

文生视频（默认 5 秒、720P、16:9）：

```powershell
python scripts/agnes_api.py video --prompt "A cinematic shot of a cat walking on the beach at sunset" --poll
```

指定时长与画幅：

```powershell
python scripts/agnes_api.py video --prompt "Neon-lit street after rain, cinematic camera move" --seconds 8 --aspect-ratio 21:9 --poll
```

图片参考视频（模式自动推断为 `reference`，提示词中用 `<Picture N>` 指代素材）：

```powershell
python scripts/agnes_api.py video --prompt "Use <Picture 1> as the character reference, natural motion" --image https://example.com/character.png --poll
```

音频参考视频（可与图片参考同时使用，最多 3 段）：

```powershell
python scripts/agnes_api.py video --prompt "Follow the rhythm of <Audio 1>, cinematic night driving" --audio https://example.com/music.mp3 --poll
```

首尾帧控制视频（模式自动推断为 `keyframe`）：

```powershell
python scripts/agnes_api.py video --prompt "Smooth transition from first frame to last frame" --first-frame https://example.com/first.png --last-frame https://example.com/last.png --poll
```

查询视频任务：

```powershell
python scripts/agnes_api.py video-get video_123456
```

Flash 版限制：`size` 固定为 `720P`，`seconds` 为 4–12 的整数，参考图片最多 5 张、参考音频最多 3 段，不支持参考视频输入。创建请求遇 `503 video_queue_full` 时脚本会自动退避重试（最多 6 次）。

默认输出会整理出常用字段，例如 `content`、`urls`、`translated_prompt`、`next_steps`，同时保留 `raw` 原始响应，方便 AI 和人工继续处理。

默认情况下，生成图片或视频后直接返回 `urls` 即可；除非用户明确要求下载、保存或检查本地文件，否则不需要把媒体下载到本地。

流式文本也会聚合输出 `content`，同时保留事件数量、是否完成和原始响应前缀，便于快速判断流式接口是否正常。

视频创建接口如果返回 `video_id`，脚本会优先使用新版 `video_id` 查询接口：`/agnesapi?video_id=...&model_name=agnes-video-2.5-flash`。如果 `video_id` 是 `task_...` 形式，脚本回退到兼容的 `task_id` 查询接口。视频完成后，脚本会从 `video_url`、`url`、`metadata.url`（2026-10-27 实测新站将 mp4 放在 `metadata.url`）或 `remixed_from_video_id` 提取可直接访问的 mp4 链接，并放入 `urls`。

注意：Agnes Responses API 的多轮函数调用目前不适合作为 Codex / Claude Code 这类 agent 的自动工具循环模型。本 skill 的脚本使用 chat completions 路径；工具调用只应视为请求结构兼容能力，而不是稳定的多轮工具执行能力。

若只想看 Agnes 原始响应，可以加：

```powershell
--raw
```

运行测试：

```powershell
python scripts/agnes_api.py smoke-test
```

默认测试覆盖文本、流式文本、工具调用请求结构和文生图，不会创建视频任务。工具调用请求有时会被 Agnes 接收但不返回 `tool_calls`，默认只输出 warning；如果你想把这种情况视为失败，可以使用：

```powershell
python scripts/agnes_api.py smoke-test --strict-tools
```

测试图生图：

```powershell
python scripts/agnes_api.py smoke-test --include-image-edit
```

单独测试某个视频能力，避免一次创建太多视频任务：

```powershell
python scripts/agnes_api.py smoke-test --video-case text
```

可选的视频测试项：

- `text`
- `keyframe`（同时测试图生图以获得两帧素材）
- `reference-image`

## 提示词语言策略

Agnes 视频生成使用英文提示词更稳定。因此本 skill 的脚本默认会检测图片/视频提示词中的非英文字符，并先调用 `agnes-3.0-flash` 翻译成英文生成提示词，再调用图片或视频 API。

翻译时会保留：

- 主体
- 场景
- 风格
- 光照
- 构图
- 镜头运动
- 动作描述
- 负面提示词或约束

如果你确实想跳过自动翻译，可以加：

```powershell
--no-translate-prompt
```

示例：

```powershell
python scripts/agnes_api.py video --prompt "中文提示词" --no-translate-prompt
```

## 测试状态

2026-10-27 全量迁移到 `https://api.agnes-ai.cn` 后，已通过真实 API 测试（新站）：

- 基础文本生成（`agnes-3.0-flash`）
- 流式文本
- Thinking 模式（默认开启与 `--no-thinking`）
- 文生图默认 2K+1:1（实测输出 2048x2048）
- 指定 2K+16:9（实测输出 2624x1472）
- 精确尺寸 1024x768 被服务端标准化为 1152x864（符合文档预期）
- 图生图
- 无效图片 URL 返回清晰 400 错误
- 中文提示词自动翻译后文生图 / 文生视频
- 文生视频完成并返回 mp4 URL（`metadata.url`，720P 4 秒）

已按官方文档实现、参数校验已本地验证，但尚未完整端到端实测（测试期间服务端持续返回 `503 video_queue_full`，免费队列饱和）：

- 首尾帧控制视频（keyframe 模式）
- 图片参考视频（reference 模式，图片）
- 音频参考视频（reference 模式，音频；从未实测）

说明：队列满为服务端瞬时拥塞，脚本已内置退避重试；建议逐个模式测试并保留 provider 错误信息。

## 仓库结构

```text
.
├── SKILL.md
├── README.md
├── README_EN.md
├── LICENSE
├── agents/
│   └── openai.yaml
├── references/
│   └── api.md
└── scripts/
    └── agnes_api.py
```

## 许可证

MIT License. See [LICENSE](LICENSE).
