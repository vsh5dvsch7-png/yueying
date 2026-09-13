<!-- mcp-name: io.github.vsh5dvsch7-png/yueying -->
# yueying — let AI watch videos

**Point Claude, Cursor or any MCP client at a video and get back a timestamped transcript plus keyframe contact sheets — offline, no API key.** Local files first; URLs (YouTube, Bilibili, Douyin, Xiaohongshu, TikTok, Vimeo, …) are videos you are entitled to process, fetched via yt-dlp at ≤720p and deleted after processing by default.

[![PyPI](https://img.shields.io/pypi/v/yueying)](https://pypi.org/project/yueying/)
[![PyPI downloads](https://img.shields.io/pypi/dm/yueying)](https://pypi.org/project/yueying/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)](pyproject.toml)
[![Add to Cursor](https://img.shields.io/badge/Add_to-Cursor-111111?logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=yueying&config=eyJjb21tYW5kIjoidXZ4IiwiYXJncyI6WyJ5dWV5aW5nIiwibWNwIl0sImVudiI6eyJQWVRIT05VVEY4IjoiMSJ9fQ==)
[![Install in VS Code](https://img.shields.io/badge/Install_in-VS_Code-0098FF?logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect?url=vscode%3Amcp%2Finstall%3F%257B%2522name%2522%253A%2522yueying%2522%252C%2522command%2522%253A%2522uvx%2522%252C%2522args%2522%253A%255B%2522yueying%2522%252C%2522mcp%2522%255D%252C%2522env%2522%253A%257B%2522PYTHONUTF8%2522%253A%25221%2522%257D%257D)

[中文说明 ↓](#中文说明)

Yueying (阅影) means "read video". One package gives you an **MCP server**, a **CLI** and an **agent skill**.

## What you get

![Claude Desktop: a YouTube link is pasted, the watch_video tool runs for about 40 seconds, and Claude answers with timestamped key points](docs/demo-desktop.gif)

*Claude Desktop with yueying connected: paste a link, wait about forty seconds, get the video back as
timestamped notes. This video ships captions, so speech recognition never ran, and the model asked for
the transcript only. Keyframes and contact sheets come back through `get_frames` when it needs to see
the screen. Demo video: [GitInGifs: Git Branches](https://www.youtube.com/watch?v=Q5OaMTd7PwM) by GitLab, CC BY.*

![A 3x3 contact sheet: nine keyframes, each with a yellow "#number mm:ss" label bottom-left](docs/demo-grid.jpg)

*Contact sheet from a 24-second demo clip (four app screenshots with Chinese narration). The yellow label on every tile is the keyframe number and timestamp; the model cites them back to you.*

The transcript of the same clip — local speech recognition, language auto-detected as Chinese:

```
[00:00] 这是阅读,一个安静的桌面小说阅读器。整本书连续滚动,按段落记住进度。
        第二个画面是桌面模式,窗口变透明,只留文字浮在桌面上。
        第三个画面是伪装皮肤,一键变成代码编辑器。
        最后是伪装成表格的样子。
```

(The app is called 月读; ASR heard the homophone 阅读. Speech recognition does that to names — the model corrects it from the on-screen text in the frames.)

Every video becomes one folder:

```
report.md          index for the model: metadata, chapters, contact sheets, keyframes, transcript
transcript.txt     paragraphs with [mm:ss] timestamps
transcript.srt     subtitles for any player
grid_01.jpg …      3x3 contact sheets, 9 keyframes each, in time order
frames/            full-size keyframes, e.g. f003_00m15s.jpg
manifest.json      machine-readable result (paths, segments, chapters, options)
```

## Why yueying

- **Captions first, Whisper only when needed.** Platform subtitles are used when they exist. Otherwise local [faster-whisper](https://github.com/SYSTRAN/faster-whisper): `large-v3-turbo` on an NVIDIA GPU, `small` on CPU, automatic CPU fallback — nothing is uploaded, no key.
- **ffmpeg bundled.** Works on Windows 11 out of the box (imageio-ffmpeg); no PATH fiddling.
- **Token-efficient.** Keyframes are taken at scene changes, near-duplicates dropped, then packed into 3x3 contact sheets with burned-in timestamps. One sheet ≈ 1–2K tokens for nine moments; one transcript with `[mm:ss]` paragraphs.
- **Chinese platforms and the rest.** Bilibili (multi-part, collections, member videos with your browser login), Douyin, Xiaohongshu — and YouTube, TikTok, Vimeo, X and every other yt-dlp site.
- **Zero API keys, zero telemetry.** The only network traffic is the video site you name and one Whisper model download. See the [privacy policy](#privacy-policy).

Benchmark: a 6-minute Bilibili video → report in ~90 s on an RTX 5060 laptop; on CPU with `model=small` expect ~1–2 min per 10 min of speech.

## Quick start

1. Install [uv](https://docs.astral.sh/uv/) (Python is not required):
   ```bash
   winget install astral-sh.uv                         # Windows
   brew install uv                                     # macOS
   curl -LsSf https://astral.sh/uv/install.sh | sh     # Linux / macOS
   ```
2. Warm up and check everything once (installs the package, probes the GPU, downloads the speech model, runs a 2-second smoke test, prints config to paste):
   ```bash
   uvx yueying mcp --setup
   ```
3. Add the server to your client (below), then ask: *"Watch C:\videos\lecture3.mp4 and turn the steps into notes"* or *"What does this video say about docker compose: https://www.bilibili.com/video/BV…"*.

### Claude Desktop

`%APPDATA%\Claude\claude_desktop_config.json` (Windows) · `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS). Fully quit and reopen Claude afterwards.

```json
{ "mcpServers": { "yueying": { "command": "uvx", "args": ["yueying", "mcp"], "env": { "PYTHONUTF8": "1" } } } }
```

Windows note: Claude Desktop does not always see your PATH — if the server fails to start ("spawn uvx ENOENT"), use the absolute path, e.g. `"command": "C:\\Users\\<you>\\.local\\bin\\uvx.exe"` (`where uvx` prints it). Logs: `%APPDATA%\Claude\logs\mcp-server-yueying.log` (`~/Library/Logs/Claude/` on macOS). Keep `wait_seconds` at its default there; see [the RUNNING rule](#the-running-rule).

### Claude Code

```bash
claude mcp add --transport stdio --scope user yueying --env PYTHONUTF8=1 -- uvx yueying mcp
```

Or drop this repo's [`.mcp.json`](.mcp.json) into a project (it ships with `"timeout": 1800000` so one `watch_video` call can wait for a long video). To raise Claude Code's tool timeout globally, set `MCP_TOOL_TIMEOUT=1800000` (ms) in your environment. The repo is also a Claude Code plugin (`.claude-plugin/plugin.json`: server + skill).

### Cursor

Click the **Add to Cursor** badge above, or put the same JSON in `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project):

```json
{ "mcpServers": { "yueying": { "command": "uvx", "args": ["yueying", "mcp"], "env": { "PYTHONUTF8": "1" } } } }
```

### Cline

MCP Servers → Configure (`cline_mcp_settings.json`). `timeout` is in seconds; the five read-only tools are safe to auto-approve. Step-by-step agent instructions: [llms-install.md](llms-install.md).

```json
{
  "mcpServers": {
    "yueying": {
      "type": "stdio",
      "command": "uvx",
      "args": ["yueying", "mcp"],
      "env": { "PYTHONUTF8": "1" },
      "timeout": 1800,
      "autoApprove": ["get_transcript", "search_transcript", "get_frames", "get_frame_at", "list_videos"]
    }
  }
}
```

### Windsurf

`~/.codeium/windsurf/mcp_config.json`:

```json
{ "mcpServers": { "yueying": { "command": "uvx", "args": ["yueying", "mcp"], "env": { "PYTHONUTF8": "1" } } } }
```

### VS Code (Copilot agent mode)

Click the **Install in VS Code** badge above, or create `.vscode/mcp.json` (note the root key `servers`):

```json
{ "servers": { "yueying": { "type": "stdio", "command": "uvx", "args": ["yueying", "mcp"], "env": { "PYTHONUTF8": "1" } } } }
```

Direct links for hosts that accept custom URL schemes: `cursor://anysphere.cursor-deeplink/mcp/install?name=yueying&config=eyJjb21tYW5kIjoidXZ4IiwiYXJncyI6WyJ5dWV5aW5nIiwibWNwIl0sImVudiI6eyJQWVRIT05VVEY4IjoiMSJ9fQ==` and `vscode:mcp/install?%7B%22name%22%3A%22yueying%22%2C%22command%22%3A%22uvx%22%2C%22args%22%3A%5B%22yueying%22%2C%22mcp%22%5D%2C%22env%22%3A%7B%22PYTHONUTF8%22%3A%221%22%7D%7D`.

### Without uv (pip / pipx) and Windows one-click

```bash
pip install yueying            # or: pipx install yueying
yueying mcp --setup            # prints a config with the absolute path of the yueying-mcp executable
```

Use that absolute path as `"command"` with no `args` (Windows: `...\Scripts\yueying-mcp.exe`; also works as `python -m yueying mcp`). Windows users without Python tooling can double-click [`install.cmd`](install.cmd) from a checkout: it creates `%LOCALAPPDATA%\yueying\venv`, installs the Claude Code skill, runs `yueying mcp --setup` and prints the JSON block with the right path.

### GPU

```bash
uvx --from "yueying[cuda]" yueying mcp     # NVIDIA: adds the CUDA runtime wheels (cuBLAS, cuDNN)
pip install "yueying[cuda]"
```

Device and model are chosen automatically (`model=auto`: large-v3-turbo on CUDA, small on CPU); if the GPU trial fails, recognition falls back to CPU by itself.

### Docker

```bash
docker build -t yueying .
docker run --rm -i -v yueying-data:/data -v "$PWD/videos:/videos:ro" yueying
```

The image is CPU-only (containers get no GPU by default), so it defaults to the `small` model.
Mount your videos read-only and give the tools container paths (`/videos/lesson.mp4`); results and
the downloaded Whisper weights live in the `/data` volume. In a client config the `command` is
`docker` and `args` are `["run", "--rm", "-i", "-v", "yueying-data:/data", "-v", "/your/videos:/videos:ro", "yueying"]`.

## Tools

| Tool | When the agent uses it | What it returns | Limits |
|---|---|---|---|
| `watch_video(video, mode="full", language="auto", model="auto", frame_interval_seconds=None, cookies_from_browser=None, output_dir=None, refresh=False, wait_seconds=45, max_chars=12000)` | First call for any video: an absolute local path or a URL. `mode`: `full` (transcript + keyframes), `transcript`, `frames`. | `DONE` overview: title, source, duration, text source, folder, files, chapters, contact-sheet ranges, transcript in `[mm:ss]` paragraphs — or `RUNNING` with stage/percent/ETA, or `ERROR` with a plain-English hint. | Blocks up to `wait_seconds` (0–1500). Transcript truncated at `max_chars` with a `get_transcript` start time. Cached per video; `refresh=true` reprocesses. |
| `get_transcript(video, start="0", end=None, format="paragraphs", max_chars=8000)` | The overview was truncated, a specific time range, or exporting subtitles (`format="srt"`). | Header + `[mm:ss]` paragraphs / `[mm:ss-mm:ss]` segments / SRT blocks; `TRUNCATED — next_start="…"` when cut. | `max_chars` 1000–100000. Times: seconds, `mm:ss`, `h:mm:ss`. |
| `search_transcript(video, query, context_seconds=15, limit=10)` | "When does he mention X?" | Hits with time, surrounding sentences, nearest keyframe number and contact-sheet number. | Space-separated terms, any matches, more terms rank higher; `limit` ≤ 50. |
| `get_frames(video, kind="grids", start=1, count=2, max_width=1280)` | See what is on screen: contact sheets first (`grids`), single keyframes (`frames`) only for detail. | Absolute path + JPEG per image, in time order; `Next: get_frames(start=…)` when more remain. | ≤ 3 images per call (default 2, keep ≤ 2 in Claude Desktop); ~150 KB per sheet at 1280 px. |
| `get_frame_at(video, time, max_width=960)` | Read code, a slide, a chart or UI at one moment. | The exact frame (extracted from the source when the local file still exists) or the nearest cached keyframe, plus the paragraphs spoken around then. | One image, ~100 KB at 960 px. |
| `list_videos(limit=20)` | The user refers to an earlier video, or to check disk use. | Table: video_id, title, duration, text source, date, size, folder; running jobs. | Instant, read-only. |

`video` for the read tools accepts the `video_id` from `watch_video`/`list_videos`, the results folder, or the same path/URL you gave `watch_video`.

### The RUNNING rule

Processing can take minutes, and most hosts cap a tool call at about a minute. So `watch_video` waits at most `wait_seconds`, then answers `RUNNING video_id=… · stage 2/4 speech recognition 40% · elapsed 46 s · est. ~1 min remaining`. The agent simply **calls `watch_video` again with the same `video`** — it re-attaches to the same job (options are ignored while it runs; `refresh=true` restarts). Recommended `wait_seconds`:

| Host | `wait_seconds` | Why |
|---|---|---|
| Claude Desktop | 45 (default) | hard ~60 s client timeout |
| Cursor | 45 (default) | 60–120 s |
| Claude Code | up to 1500 | with `.mcp.json` `timeout` / `MCP_TOOL_TIMEOUT` = 1800000 ms |
| Cline | up to 1500 | with `"timeout": 1800` (s) |

Progress notifications are sent every 1.5 s for hosts that display them. One video is processed at a time per server; extra requests queue.

**First run:** the first speech recognition downloads a Whisper model once (~480 MB `small` on CPU, ~1.6 GB `large-v3-turbo` on GPU). `uvx yueying mcp --setup` does this ahead of time; otherwise the `RUNNING` line says "first run downloads ~… this can take several minutes".

## Where files go

Root: `$YUEYING_OUT_DIR` if set, else `~/yueying_out`. One entry per video, named `<slug>-<video_id>` (`yt-<id>`, `bili-<BV>`, or the file name) — never renamed; the title lives in `manifest.json`.

```
~/yueying_out/
└── bili-BV1xx-3f9a2c1e/
    ├── report.md  transcript.txt  transcript.srt  manifest.json
    ├── grid_01.jpg … grid_07.jpg
    ├── frames/            f001_00m02s.jpg …   (+ frames/extra/ for get_frame_at)
    ├── .job               only while a job runs
    └── _download/         only with YUEYING_KEEP_SOURCE=1
```

| Environment variable | Meaning | Default |
|---|---|---|
| `YUEYING_OUT_DIR` | root folder for results (absolute, `~` ok) | `~/yueying_out` |
| `YUEYING_MODEL` | default for the `model` parameter | `auto` |
| `YUEYING_DEVICE` | `auto` / `cuda` / `cpu` | `auto` |
| `YUEYING_LANG` | language of report.md written by the server (`en`/`zh`) | `en` |
| `YUEYING_COOKIES_FROM_BROWSER` | default browser for cookies (`chrome`, `edge`, `firefox`, …) | unset |
| `YUEYING_KEEP_SOURCE` | `1` keeps the downloaded ≤720p source in `_download/` (enables exact-moment frames for URLs) | unset |
| `YUEYING_MAX_JOBS` | pipelines running at once per server | `1` |
| `YUEYING_JOB_TIMEOUT` | hard limit per video, seconds | `7200` |
| `HF_HOME` | Hugging Face cache (Whisper weights live here) | HF default |
| `HF_ENDPOINT` | mirror, e.g. `https://hf-mirror.com` | huggingface.co |
| `PYTHONUTF8` | set to `1` on Windows to avoid mojibake | — |

Disk budget: ≈ 25 MB per hour of video; 300–600 MB/h more with `YUEYING_KEEP_SOURCE=1`. Nothing is deleted automatically — `list_videos` shows sizes; delete a folder to free space; `watch_video(refresh=true)` reprocesses one video. Editing a local file changes its size/mtime and therefore gets a new entry.

## Supported sources

- **Local files:** anything ffmpeg reads — mp4, mkv, mov, webm, avi, flv, ts, and audio (mp3, m4a, wav, …). Audio-only input gives a transcript without frames.
- **URLs:** every site yt-dlp supports. Fetched at ≤720p and deleted after processing unless `YUEYING_KEEP_SOURCE=1`.
- **Bilibili:** without login Bilibili serves 480p — enough for slides and code. For HD or member-only videos pass `cookies_from_browser="chrome"` (or `edge`, `firefox`, `brave`, `chromium`, `safari`); on Windows close Chrome first, it locks its cookie database. Multi-part videos and collections: `p=` links are separate entries; the CLI's `--all` processes them all.
- **Not for** live streams or images. Short links (`b23.tv`, `v.douyin.com`) are processed but not de-duplicated against their long form (the server never resolves URLs itself).

## Also a CLI and an agent skill

```bash
yueying video.mp4
yueying "https://www.bilibili.com/video/BVxxxx" --ui-lang en
yueying "https://www.youtube.com/watch?v=xxxx" --out ./notes/xxx
yueying lesson1.mp4 lesson2.mp4 "https://www.bilibili.com/video/BVyyyy"   # several at once, one folder each + index.md
yueying "https://www.bilibili.com/video/BVxxxx" --all                      # every part of a multi-part video / collection
yueying --install-skill                                                    # Claude Code skill -> ~/.claude/skills/yueying
```

Default output: `./yueying_out/<name>/` (parent folder when several inputs). CLI log lines and report.md are Chinese by default (`--ui-lang en` for English); 0.3 will flip the default to English.

| Flag | Meaning |
|---|---|
| `--out DIR` | output folder (default `./yueying_out/<name>`; the parent folder when several inputs) |
| `--all` | when the URL is a Bilibili multi-part video / collection / playlist, process every entry (default: only the first) |
| `--lang zh` | spoken language code (`zh`, `en`, `ja`, …); default auto-detect |
| `--model auto` | Whisper model: `auto` / `tiny` / `base` / `small` / `medium` / `large-v3` / `large-v3-turbo` (CLI default). `auto` = large-v3-turbo on an NVIDIA GPU, small on CPU |
| `--device cpu` | force CPU (`auto` / `cuda` / `cpu`) |
| `--interval 3` | roughly one keyframe every N seconds. Default by duration: 2 s under 1 min, 3 s under 3 min, 6 s under 10 min, 12 s under 30 min, 20 s beyond |
| `--frames 30` | maximum number of keyframes (default by duration, cap 150; 300 with `--interval`) |
| `--scene 0.2` | scene-change sensitivity 0–1, lower = more sensitive (default 0.3) |
| `--no-dedupe` | keep frames that are almost identical to the previous one (default drops them: < 2 % of thumbnail pixels changed) |
| `--no-asr` | no speech recognition even without subtitles (pictures only) |
| `--no-frames` | no keyframes (text only) |
| `--force-asr` | run speech recognition even when subtitles exist |
| `--cookies-from-browser chrome` | download with your browser login (Bilibili HD / member videos, sign-in-gated YouTube) |
| `--keep` | keep the downloaded source video |
| `--ui-lang en` | language of report.md headings and labels: `zh` (default) or `en` |
| `--json` | print one line of manifest JSON at the end (for scripts) |
| `--install-skill` | install the agent skill into `~/.claude/skills/yueying` |
| `mcp` | run the MCP server (`mcp --setup`, `mcp --check`, `mcp --version`) |

The skill (`src/yueying/skill/SKILL.md`) tells a coding agent to prefer the MCP tools when present and otherwise run the CLI and read `report.md` plus the contact sheets. Tools that support the Agent Skills standard can copy `~/.claude/skills/yueying/SKILL.md` into their own skills folder.

## Compared with similar projects (September 2026)

| | yueying | claude-video | claude-real-video | mcp-video-analyzer |
|---|---|---|---|---|
| MCP server | yes | no (skill only) | no (skill) | yes (Node) |
| Offline speech recognition | yes — subtitles first, local faster-whisper otherwise | cloud Whisper fallback | ASR-first | whisper installed separately |
| GPU auto-detect + CPU fallback | yes | – | – | – |
| ffmpeg bundled | yes | – | manual ffmpeg | – |
| Windows tested | yes (Windows 11) | – | – | – |
| Contact sheets (3x3) | yes | – | – | – |
| Burned-in timestamps on frames | yes | – | – | – |
| Bilibili / Douyin / Xiaohongshu | yes | – | – | – |

"–" means the project did not advertise the feature when we looked; check their READMEs, they may have moved on.

## Privacy policy

yueying collects nothing and has no telemetry, analytics, crash reporting or update checks. All processing is local. The only network connections are (1) to the video site of the URL you pass, through yt-dlp, and (2) to Hugging Face (or `HF_ENDPOINT`) to download a Whisper model once. Outputs are stored in your folder until you delete them. The transcript and any frames you request are sent only to the model your MCP client is configured to use — that transfer is governed by your client's and provider's terms, not by yueying. Questions: [GitHub issues](https://github.com/vsh5dvsch7-png/yueying/issues). Full text: [docs/privacy.md](docs/privacy.md).

## Troubleshooting / FAQ

- **"No result received" in Claude Desktop.** Keep `wait_seconds` at 45 (the agent then re-calls `watch_video`), and run `uvx yueying mcp --setup` once so the first call is not also the model download.
- **`spawn uvx ENOENT` / server fails to start.** The host cannot see your PATH: use the absolute path to `uvx` (`where uvx` / `which uvx`) or to `yueying-mcp` as `"command"`.
- **Console windows flash on Windows.** Update to 0.2.0+: child processes are started without a window. If you still see them, you are running an old install (`uv cache clean yueying`).
- **Mojibake / `?????` in titles.** Add `"env": { "PYTHONUTF8": "1" }` to the server config (all snippets above include it).
- **Bilibili error 412 / "-352".** The site wants a login: `cookies_from_browser="edge"` or `"chrome"` (close Chrome first on Windows).
- **YouTube "Sign in to confirm you're not a bot".** Same fix: `cookies_from_browser`. Also try updating yt-dlp: `uv cache clean yueying` or `pip install -U yt-dlp`.
- **Slow on CPU.** `model="small"` is already the automatic choice without an NVIDIA GPU; use `mode="frames"` when only the pictures matter, or `mode="transcript"` to skip keyframes.
- **Names, numbers and code are wrong in the transcript.** Expected with any ASR — the agent is told to trust on-screen text; ask it to `get_frame_at` the moment.
- **Model download is slow or blocked (mainland China).** Set `HF_ENDPOINT=https://hf-mirror.com` in the server `env` (the CLI switches to the mirror automatically when huggingface.co is unreachable).
- **GPU error (CUDA / cuDNN / out of memory).** `model="small"` or `YUEYING_DEVICE=cpu`; install the CUDA wheels with `yueying[cuda]`.
- **Want to reprocess with different settings.** `watch_video(video=…, refresh=true, …)` — it kills a running job for that video, deletes the entry and starts again.

## How it works

```
video / URL ──► yt-dlp (≤720p) + platform subtitles
            ──► ffmpeg 16 kHz audio ──► faster-whisper (skipped when subtitles exist)
            ──► ffmpeg scene detection ──► keyframes (near-duplicates dropped)
            ──► Pillow: burn "#n mm:ss", pack 3x3 contact sheets
            ──► report.md · transcript.txt · transcript.srt · manifest.json

MCP client ──stdio──► mcp_server.py ──spawns──► python -m yueying.cli <video> --json --ui-lang en
                       │  parses the child's progress lines, long-polls, caches per video
                       └─► store.py (cache keys / folders) · query.py (paging, search, frames)
```

The server process never loads yt-dlp, Whisper or CUDA itself; all heavy work runs in a child process that is killed with the server. Everything lives in `src/yueying/`:

| File | Responsibility |
|---|---|
| `cli.py` | command-line entry; runs the pipeline; `mcp` subcommand dispatch; `--install-skill` |
| `mcp_server.py` | the MCP server: six tools, job runner, progress parsing, DONE/RUNNING/ERROR rendering, `--setup` |
| `store.py` | output root, canonical URLs, per-video cache keys and folder names, `.job` markers |
| `query.py` | pure functions over a manifest: transcript paging, search, nearest frame, contact-sheet ranges, image shrinking |
| `models.py` | Whisper model names, sizes and Hugging Face repos (no heavy imports) |
| `download.py` | yt-dlp download, subtitle language choice, playlist/collection listing |
| `ffm.py` | ffmpeg wrapper: probe, audio extraction, embedded subtitles, timeouts |
| `subs.py` | srt / vtt / Bilibili JSON subtitle parsing, YouTube auto-caption de-duplication |
| `asr.py` | faster-whisper transcription, GPU/CPU selection, `auto` model, fallbacks |
| `frames.py` | scene detection, frame timing, extraction, de-duplication, timestamp burn-in, contact sheets |
| `report.py` | report.md, transcript files, manifest.json (zh/en labels), manifest loading |
| `skill/SKILL.md` | the agent skill |

## Roadmap

- **next** — `.mcpb` one-click bundle for Claude Desktop, Smithery listing.
- **0.3** — `forget_video` / prune tools, English as the CLI default report language, `--all` (playlists) over MCP.

## Credits

- [faster-whisper](https://github.com/SYSTRAN/faster-whisper), [yt-dlp](https://github.com/yt-dlp/yt-dlp), [imageio-ffmpeg](https://github.com/imageio/imageio-ffmpeg), [Pillow](https://python-pillow.org/)
- Burned-in timestamps on contact sheets follow [video-vision-mcp](https://github.com/OAMaestro/video-vision-mcp); duration-based frame intervals follow [video-analyzer-skill](https://github.com/bsisduck/video-analyzer-skill)

## License

MIT — see [LICENSE](LICENSE).

---

## 中文说明

**阅影（yueying）让 AI 看懂视频。** 给 Claude Desktop、Claude Code、Cursor 等支持 MCP 的工具一个本地视频文件或视频链接（B站 / YouTube / 抖音 / 小红书 …），它就能拿到带时间戳的文字稿和关键帧九宫格：字幕优先，没有字幕就本地 faster-whisper 识别，全程离线，不上传、不要 API key。链接通过 yt-dlp 以 ≤720p 下载，处理完默认删除原视频。完整文档见上方英文部分。

### 安装

```bash
# 1. 装 uv（不需要先装 Python）
winget install astral-sh.uv          # Windows；macOS: brew install uv
# 2. 预热：安装、探测显卡、下载模型、跑 2 秒冒烟测试、打印配置
uvx yueying mcp --setup
```

不用 uv：`pip install yueying`（有 NVIDIA 显卡再加 `pip install "yueying[cuda]"`），然后 `yueying mcp --setup` 会打印 `yueying-mcp` 的绝对路径。Windows 也可以下载仓库后双击 `install.cmd`，它会建独立环境、装 Claude Code 技能、跑 `--setup` 并打印可粘贴的配置。

### 配置

Claude Desktop（`%APPDATA%\Claude\claude_desktop_config.json`，改完完全退出再打开；Windows 下 `uvx` 找不到就写绝对路径，如 `C:\\Users\\<你>\\.local\\bin\\uvx.exe`）、Cursor（`~/.cursor/mcp.json`）、Windsurf（`~/.codeium/windsurf/mcp_config.json`）都是同一段：

```json
{ "mcpServers": { "yueying": { "command": "uvx", "args": ["yueying", "mcp"], "env": { "PYTHONUTF8": "1" } } } }
```

Claude Code 一行：

```bash
claude mcp add --transport stdio --scope user yueying --env PYTHONUTF8=1 -- uvx yueying mcp
```

Cline 在同一段里加 `"type": "stdio"`、`"timeout": 1800`，并把 `get_transcript`、`search_transcript`、`get_frames`、`get_frame_at`、`list_videos` 放进 `autoApprove`。VS Code 的 `.vscode/mcp.json` 根键是 `servers` 并加 `"type": "stdio"`。

### 六个工具

| 工具 | 用途 |
|---|---|
| `watch_video(video, mode, language, model, frame_interval_seconds, cookies_from_browser, output_dir, refresh, wait_seconds, max_chars)` | 看一个视频：本地绝对路径或链接。返回 `DONE`（概览 + 文字稿）、`RUNNING`（进度，用同一个 `video` 再调一次即可继续等）或 `ERROR`（英文提示）。结果按视频缓存，`refresh=true` 重做 |
| `get_transcript(video, start, end, format, max_chars)` | 按时间段读文字稿；`format="srt"` 导出字幕 |
| `search_transcript(video, query, context_seconds, limit)` | 找「哪里提到了 X」，给出时间、上下文、最近的关键帧号和九宫格号 |
| `get_frames(video, kind, start, count, max_width)` | 看画面：先看九宫格（`grids`），需要细节再看单帧（`frames`）；每次最多 3 张 |
| `get_frame_at(video, time, max_width)` | 看某一时刻：代码、PPT、图表、界面；本地文件还在就精确抽帧 |
| `list_videos(limit)` | 列出处理过的视频（video_id、标题、时长、文字来源、日期、大小、目录）和正在跑的任务 |

Claude Desktop / Cursor 里 `wait_seconds` 保持默认 45；Claude Code / Cline 配好 timeout 后可以给到 1500，一次调用就等到结果。第一次语音识别要下载模型（CPU 约 480 MB 的 small，显卡约 1.6 GB 的 large-v3-turbo），`--setup` 会提前下好。

### 输出目录与环境变量

结果在 `~/yueying_out/<名字>-<video_id>/`（`YUEYING_OUT_DIR` 可改），每个视频一个文件夹：`report.md`、`transcript.txt`、`transcript.srt`、`grid_01.jpg …`、`frames/`、`manifest.json`。约每小时视频 25 MB；不会自动删，删文件夹即可。常用环境变量：`YUEYING_OUT_DIR`（输出根目录）、`YUEYING_MODEL`（默认 `auto`：有 NVIDIA 显卡用 large-v3-turbo，否则 small）、`YUEYING_DEVICE`（`auto`/`cuda`/`cpu`）、`YUEYING_LANG`（服务端 report.md 语言，默认 `en`，中文写 `zh`）、`YUEYING_KEEP_SOURCE=1`（保留下载的原视频）、`YUEYING_JOB_TIMEOUT`（单个视频上限秒数，默认 7200）、`PYTHONUTF8=1`（Windows 防乱码）。

### B站 cookie

不登录 B站 只给 480p，看 PPT 和代码够用；高清或会员视频传 `cookies_from_browser="chrome"`（或 `edge`），Windows 下先关掉 Chrome，否则读不到 cookie 数据库。遇到 412 / -352 也是同样的处理。

### 国内镜像

下载模型慢或被墙：在服务器配置的 `env` 里加 `"HF_ENDPOINT": "https://hf-mirror.com"`。命令行版在 huggingface.co 连不上时会自动切到镜像。

### 命令行用法

```bash
yueying 视频.mp4
yueying "https://www.bilibili.com/video/BVxxxx"
yueying "https://www.youtube.com/watch?v=xxxx" --out ./notes/xxx
yueying 第1课.mp4 第2课.mp4 "https://www.bilibili.com/video/BVyyyy"   # 多个一起，各出各的文件夹 + index.md
yueying "https://www.bilibili.com/video/BVxxxx" --all                      # B站 分 P / 合集 全部处理
yueying --install-skill                                                    # 装 Claude Code 技能
```

默认输出 `./yueying_out/<视频名>/`，日志和 report.md 默认中文（`--ui-lang en` 切英文）。常用参数：`--lang zh` 指定语言、`--model small` 换小模型（`auto` 自动选）、`--device cpu`、`--interval 3` 抽帧间隔、`--frames 30` 最多帧数、`--scene 0.2` 场景灵敏度、`--no-dedupe` 不去重、`--no-asr` 只要画面、`--no-frames` 只要文字、`--force-asr` 有字幕也识别、`--cookies-from-browser chrome`、`--keep` 保留原视频、`--json` 末尾打印一行 manifest。完整说明见上方英文表格。
