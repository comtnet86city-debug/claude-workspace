# CLAUDE.md — AI 工作區設定文件

**GitHub:** `comtnet86city-debug/claude-workspace`
**本地路徑:** `~/CLAUDE.md`

此檔案同時扮演兩個角色：
1. **Claude Code 指引** — Claude Code CLI 自動讀取，提供工作區的環境、腳本用法與架構說明
2. **GitHub 備份** — 透過 `~/` 的 git repo（追蹤 `CLAUDE.md`、`README.md`、`.gitignore`）備份至 `claude-workspace`

更新後推送：`git -C ~ add CLAUDE.md README.md && git -C ~ commit -m "..." && git -C ~ push origin master`

---

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

This is a personal AI agent / automation workspace on Ubuntu. Scripts are organized under `~/ai_projects/`.

### Virtual Environments

| Path | Key Packages | Use For |
|------|-------------|---------|
| `~/ai_agent_env/` | crewai, torch 2.11+cu128, faster-whisper, langchain, python-pptx, openpyxl, paramiko, chromadb, duckduckgo-search | All agent scripts |
| `~/marker_env/` | marker-pdf 1.10 | PDF 精準解析（由 ingest.py 以 subprocess 呼叫） |
| `~/comfyui_env/` | torch 2.11+cu128, ComfyUI deps | ComfyUI Stable Diffusion |

Activate before running any script:
```bash
source ~/ai_agent_env/bin/activate
```

### LLM Backend

Ollama runs locally at `http://localhost:11434`. GTX 1650 has 4GB VRAM — models over 4GB spill to RAM (slower).

| Model | Size | Runs On | Used For |
|-------|------|---------|---------|
| `hermes3:3b` | 2GB | **Full GPU** | server_monitor scripts (tool-use), preloaded @reboot |
| `qwen3.6:latest` | 5.2GB | GPU+CPU hybrid | PPT / Excel / RAG (main workhorse), preloaded @reboot |
| `qwen3.6:27b-q4_K_M` | 17GB | CPU only | complex planning (slow, use sparingly) |
| `mxbai-embed-large` | 669MB | GPU | RAG embeddings only |
| `qwen2.5-coder:7b` | 4.7GB | GPU+CPU hybrid | code generation / debugging |
| `qwen3:8b` | 5.2GB | GPU+CPU hybrid | general chat alternative to qwen3.6 |
| `hermes3:latest` | 4.7GB | GPU+CPU hybrid | larger hermes variant (use hermes3:3b for tool-use) |

Check Ollama status: `ollama list`

### External Services (Docker, auto-start @reboot)

| Service | Port | URL | Purpose |
|---------|------|-----|---------|
| Open-WebUI | 8080 | http://localhost:8080 | Ollama 全模型對話 UI，支援 RAG |
| ComfyUI | 8188 | http://localhost:8188 | Stable Diffusion 圖片生成（SD 1.5 模型已就緒） |
| n8n | 5678 | http://localhost:5678 | 視覺化工作流自動化，可串接 Hermes API |
| SearXNG | 8888 | http://localhost:8888 | 聚合搜尋引擎，已整合為 Agent 預設搜尋後端 |
| 設計協作平台 | 9001 | http://localhost:9001 | Penpot 開源設計工具（Figma 替代），介面繁體中文，Docker Compose: `~/penpot/` |
| 設計協作倉庫 | 3001 | http://localhost:3001 | Gitea 設計檔案版本管理，介面繁體中文，Docker Compose: `~/gitea/` |

Start all services: `start-all`
Start ComfyUI only: `bash ~/ComfyUI/start.sh`

### OpenClaw Gateway

| User | Version | Port | Binary |
|------|---------|------|--------|
| `jwd` | `2026.5.7` | 18789 | `~/.local/bin/openclaw` → `~/.nvm/versions/node/v24.15.0/bin/openclaw` |
| `juser1` | `2026.5.6` | 18790 | `~/.npm-global/bin/openclaw` |

```bash
# 狀態 / 重啟 (jwd)
openclaw --version
openclaw gateway status
openclaw gateway restart

# 狀態 / 重啟 (juser1)
sudo -u juser1 bash -i -c 'openclaw gateway status'

# 更新版本（更新後重啟 gateway）
npm install -g openclaw@latest          # 更新 jwd (nvm)
sudo bash ~/setup_openclaw_juser1.sh    # 重新安裝 juser1
```

**jwd config** (`~/.openclaw/openclaw.json`):
- primary: `hermes3:3b` → fallbacks: `qwen3.6 → gemini-2.5-flash → qwen2.5-coder → gpt-5.4 → gpt-5.5`
- `compaction.reserveTokensFloor: 4096`、`thinkingDefault: off`、`sandbox: off`
- Google Gemini 2.5 Flash、OpenAI Codex gpt-5.4/5.5、SearXNG 網路搜尋已啟用

**juser1 config** (`/home/juser1/.openclaw/openclaw.json`):
- primary: `hermes3:3b` → fallbacks: `qwen2.5-coder → qwen3.6 → qwen3:8b`
- `sandbox: all`（所有操作沙箱隔離）、`tools.deny: [group:web, browser]`（網路工具停用）

## Running Scripts

```bash
source ~/ai_agent_env/bin/activate

# PPT generation (AI, structured, CrewAI + Pydantic)
python ~/ai_projects/ppt/final_agent.py "淡江大橋"
python ~/ai_projects/ppt/final_agent.py "淡江大橋" --slides 5
python ~/ai_projects/ppt/final_agent.py "淡江大橋" --output ~/桌面/output.pptx

# PPT generation (static, hardcoded web design proposal)
python ~/ai_projects/ppt/make_webdesign_ppt.py

# Server monitor — Qwen agent with SSH disk/health tools + web search (SearXNG primary)
cd ~/ai_projects/server_monitor && python agent.py

# Server monitor — Hermes3 agent (stricter tool-use enforcement)
cd ~/ai_projects/server_monitor && python hermes_agent.py

# Excel generation (AI-powered, CrewAI + Pydantic structured data)
python ~/ai_projects/excel/generate_excel.py --topic "季度業績報告" --rows 10
python ~/ai_projects/excel/generate_excel.py --output ~/桌面/sales.xlsx

# RAG knowledge base — ingest documents first, then query
python ~/ai_projects/rag/ingest.py ~/ai_projects/         # 嵌入所有 .py/.md/.pdf 文件（marker-pdf 精準解析 PDF）
python ~/ai_projects/rag/ingest.py ~/doc.pdf --no-marker  # 強制使用 pypdf（快速）
python ~/ai_projects/rag/query.py "CrewAI 如何定義工具？" # 查詢

# Whisper 語音轉文字
python ~/ai_projects/whisper/transcribe_agent.py ~/桌面/meeting.mp3
python ~/ai_projects/whisper/transcribe_agent.py ~/桌面/video.mp4 --summarize  # 含 AI 摘要
# 首次執行自動下載 Whisper medium 模型（~1.5GB）

# ComfyUI 圖片生成（需先啟動 ComfyUI）
bash ~/ComfyUI/start.sh
python ~/ai_projects/comfy/generate_image.py "a futuristic bridge at sunset, photorealistic"
python ~/ai_projects/comfy/generate_image.py "台灣101夜景" --translate  # 中文自動翻成英文

# TTS 語音合成（edge-tts，繁體中文）
python ~/ai_projects/tts/speak.py "要朗讀的文字"
python ~/ai_projects/tts/speak.py "要朗讀的文字" --voice zh-TW-YunJheNeural
python ~/ai_projects/tts/speak.py "要朗讀的文字" --output ~/桌面/output.mp3
python ~/ai_projects/tts/speak.py --list-voices   # 列出可用聲音

# Vision 圖片分析（moondream 辨識 + qwen3.6 繁中解說）
python ~/ai_projects/vision/describe_image.py ~/桌面/photo.jpg
python ~/ai_projects/vision/describe_image.py ~/桌面/photo.jpg --question "圖中有幾個人？"
python ~/ai_projects/vision/describe_image.py ~/桌面/photo.jpg --speak  # 朗讀結果（需 edge-tts）
python ~/ai_projects/vision/ocpaste.py                 # 從剪貼簿讀圖 → 分析 → 送 OpenClaw
python ~/ai_projects/vision/ocpaste.py ~/桌面/photo.jpg  # 指定檔案
python ~/ai_projects/vision/ocpaste.py --screenshot    # 截圖選取區域（需 scrot）
python ~/ai_projects/vision/ocpaste.py --no-agent      # 只分析，不送 OpenClaw

# Claude Code Bridge（OpenAI-compatible API，供 Open-WebUI 使用）
bash ~/ai_projects/claude_bridge/start.sh   # 啟動（port 19001）
# 模型：claude-code（完整工具）、claude-code-no-tools（純對話）

# OpenClaw Proxy（OpenAI-compatible，轉發至 GPT-5.x / Gemini）
source ~/ai_agent_env/bin/activate && python ~/ai_projects/openclaw-proxy/server.py  # port 11435
# 模型：openai-codex/gpt-5.5、openai-codex/gpt-5.4、google/gemini-2.5-flash（支援圖片）

# Hermes Dashboard (web UI at http://localhost:7860)
bash ~/hermes-dashboard/start.sh   # 啟動（已設 @reboot 自動啟動）
start-all                          # 啟動全部服務

# Health monitoring (also runs automatically via crontab every 30 min)
bash ~/ai_projects/monitor_health.sh
cat ~/ai_projects/logs/health_$(date +%Y%m).log   # 查看日誌
```

All scripts save output to `~/桌面/`.

## Project Structure

```
~/ai_projects/
├── ppt/
│   ├── final_agent.py        # AI PPT (Pydantic structured output, argparse topic)
│   └── make_webdesign_ppt.py # Static web design proposal (9 slides)
├── server_monitor/
│   ├── tools.py              # Tools: disk_tool, server_health_tool, local_system_tool,
│   │                         #        web_search_tool (SearXNG+DDG), whisper_tool, image_gen_tool
│   ├── agent.py              # Qwen server monitor agent
│   └── hermes_agent.py       # Hermes3 server monitor agent
├── excel/
│   └── generate_excel.py     # AI-powered Excel (CrewAI + Pydantic, --topic, --rows)
├── rag/
│   ├── ingest.py             # Ingest docs → ChromaDB (marker-pdf for PDF, fallback pypdf)
│   ├── query.py              # RAG query agent (CrewAI + kb_search tool)
│   └── chroma_db/            # Persistent vector store (auto-created on first ingest)
├── whisper/
│   └── transcribe_agent.py   # Whisper speech-to-text agent (--summarize for AI summary)
├── comfy/
│   └── generate_image.py     # ComfyUI Stable Diffusion agent (--translate for Chinese prompt)
├── tts/
│   └── speak.py              # edge-tts 繁體中文語音合成（--voice/--output/--list-voices）
├── vision/
│   ├── describe_image.py     # 圖片分析：moondream（英文）+ qwen3.6（繁中）+ --speak
│   └── ocpaste.py            # 剪貼簿/截圖 → moondream → qwen3.6 → OpenClaw agent
├── claude_bridge/
│   ├── app.py                # Claude Code CLI → OpenAI-compatible API (port 19001)
│   └── start.sh              # 啟動腳本（nohup uvicorn，PID 存 /tmp/claude_bridge.pid）
├── openclaw-proxy/
│   └── server.py             # OpenClaw → OpenAI-compatible proxy (port 11435)
│                             # 模型：gpt-5.5、gpt-5.4、gemini-2.5-flash（支援圖片輸入）
├── logs/
│   ├── health_YYYYMM.log     # Monthly health records (crontab every 30 min)
│   ├── alerts.log            # Threshold alerts (CPU>90%, RAM>85%, GPU>80°C, VRAM>95%)
│   └── start-all.log         # Service startup log (@reboot)
├── system_configs/
│   ├── ollama_override.conf  # Ollama systemd override
│   └── apply_gpu_split.sh    # GPU split setup: Intel→XRDP display, NVIDIA→AI compute
├── preload_models.sh         # @reboot: preload hermes3:3b + qwen3.6:latest
├── monitor_health.sh         # Health monitoring (crontab: */30 * * * *)
├── README.md                 # 功能模組總覽、快速開始、相關 repo
└── CLAUDE.md                 # 模組總覽、執行指令、架構說明
                              # Monitors: CPU/RAM/Disk/GPU + Ollama/Dashboard/WebUI/n8n/SearXNG

~/hermes-dashboard/           # FastAPI web UI — http://localhost:7860
├── main.py                   # API server (8 scripts, /api/services, /api/b44/*)
├── start.sh                  # Launch script (@reboot via crontab)
├── static/index.html         # Alpine.js SPA (7 tabs: 儀表板/Agents/模型/檔案/日誌/健康/外部服務)
├── README.md                 # 功能簡介、啟動方式、B44 API 說明
└── CLAUDE.md                 # API 端點、頁籤說明、依賴與注意事項

~/ComfyUI/                    # Stable Diffusion
├── main.py                   # ComfyUI server (port 8188)
├── start.sh                  # Start script (auto-detects if already running)
├── download_model.sh         # Download SD 1.5 (already done, 4.27GB)
└── models/checkpoints/       # v1-5-pruned-emaonly.safetensors (already downloaded)

~/penpot/                     # 設計協作平台（Penpot，port 9001）
├── docker-compose.yml        # 5 containers: frontend/backend/exporter/postgres/redis
├── index.html                # 修改版首頁（標題改為繁中），bind mount 覆蓋容器內檔案
├── README.md                 # 容器架構、中文化說明、相關 repo
└── CLAUDE.md                 # 容器清單、中文化設定、Volume 說明

~/gitea/                      # 設計協作倉庫（Gitea，port 3001，SSH: 2222）
├── docker-compose.yml        # SQLite 後端，DEFAULT_LOCALE=zh-TW，APP_NAME=設計協作倉庫
├── README.md                 # 設定重點、SSH 使用、相關 repo
└── CLAUDE.md                 # 環境變數、SSH 設定、升級指引

~/bin/                        # 個人工具腳本（已加入 $PATH）
├── start-all                 # 啟動所有 AI 服務
├── hm                        # Hermes 模型快速切換（本地 + 雲端）
├── b44-info                  # Hermes Dashboard API Token 與連線資訊
├── ocpaste                   # 剪貼簿/截圖 → AI 分析 → OpenClaw
├── clipboard-png-daemon      # X11 剪貼簿 BMP→PNG 自動轉換
├── start-clipboard-daemon.sh # 啟動 clipboard-png-daemon
├── README.md                 # 腳本清單、快速示例、相關 repo
└── CLAUDE.md                 # 所有腳本用法、模型速查表
```

## Git Repositories (GitHub: comtnet86city-debug)

| 本地路徑 | GitHub Repo | README | CLAUDE.md |
|----------|-------------|--------|-----------|
| `~/ai_projects/` | `comtnet86city-debug/ai_projects` | ✓ | 模組總覽、執行指令、架構說明、環境變數、參見 |
| `~/hermes-dashboard/` | `comtnet86city-debug/hermes-dashboard` | ✓ | API 端點完整列表、7 頁籤說明、依賴、注意事項、參見 |
| `~/penpot/` | `comtnet86city-debug/penpot` | ✓ | 容器清單、中文化設定、Volume、升級確認步驟、參見 |
| `~/gitea/` | `comtnet86city-debug/gitea` | ✓ | 環境變數、SSH 設定、app.ini 查看、升級指引、參見 |
| `~/bin/` | `comtnet86city-debug/bin` | ✓ | 所有腳本用法、hm 模型速查表、備援鏈說明、參見 |
| `~/` | `comtnet86city-debug/claude-workspace` | ✓ | 本檔即文件，各 repo CLAUDE.md 均設有參見連結 |

Push 指令：`git push origin master`（各 repo remote 均已設定 PAT）

## Code Architecture

### CrewAI Pattern

Scripts follow a three-layer pattern:
1. **LLM init** — `LLM(model="ollama/qwen3.6:latest", base_url="http://localhost:11434")`
2. **Agent definition** — role, goal, backstory, optional `tools=[]`
3. **Task + Crew** — task binds to agent via `agent=`, `output_pydantic=` enforces structured JSON output

### Structured Output

`ppt/final_agent.py` is the reference implementation: define Pydantic models (`Slide`, `PresentationModel`) and pass `output_pydantic=PresentationModel` to `Task`. Access results via `result.pydantic.slides` rather than parsing raw strings.

### Custom Tools (`server_monitor/tools.py`)

Six tools available, pass any combination via `tools=[...]`:
- `disk_tool` — SSH 磁碟空間（逗號分隔 IP）
- `server_health_tool` — SSH 完整報告（CPU/RAM/磁碟/Top3 程序）
- `local_system_tool` — 本機狀態（CPU/RAM/磁碟/GPU VRAM）
- `web_search_tool` — SearXNG 優先 + DuckDuckGo 備援（自動 fallback）
- `whisper_tool` — 本地語音轉文字（faster-whisper medium，CUDA 加速）
- `image_gen_tool` — ComfyUI Stable Diffusion 圖片生成（需 ComfyUI 在線）

SSH connects with `paramiko.RejectPolicy()` (safe). Credentials auto-detected in this order:
1. Auto-discover `~/.ssh/id_ed25519` or `~/.ssh/id_rsa`
2. `export SSH_KEY=/path/to/key`
3. `export SSH_PASS=yourpassword`
4. `export SSH_USER=ubuntu SSH_PORT=2222` — override defaults (root / 22)

### RAG Knowledge Base (`rag/`)

Uses `mxbai-embed-large` for embeddings + ChromaDB for storage. PDF parsing uses marker-pdf (ML-based, handles scanned/complex layouts) via subprocess to `~/marker_env/`, falls back to pypdf.

### Whisper Transcription (`whisper/`)

Uses `faster-whisper` with CUDA float16 for GPU-accelerated transcription. Model (`medium`) auto-downloaded on first use (~1.5GB). Supports `.mp3/.wav/.mp4/.m4a/.ogg/.flac/.webm`. Optional `--summarize` flag adds AI-generated summary via CrewAI.

### ComfyUI Image Generation (`comfy/`)

Calls ComfyUI REST API (port 8188) with SD 1.5 checkpoint. Optional `--translate` flag uses LLM to convert Chinese prompts to Stable Diffusion English prompts. Output saved to `~/桌面/`.

### Hermes Dashboard (`~/hermes-dashboard/`)

FastAPI backend at port 7860. 7-tab Alpine.js SPA:
- **儀表板** — system gauges, quick-launch, recent files
- **Agents** — 8 scripts with parameter input modal, live terminal
- **模型** — Ollama model load/unload
- **輸出檔案** — Desktop file browser + download
- **執行日誌** — run history
- **健康日誌** — 30-min health log + alerts
- **外部服務** — Open-WebUI / ComfyUI / n8n / SearXNG / 設計協作平台 / 設計協作倉庫 status + start buttons

Key APIs: `/api/run?script=X&extra=Y`, `/api/services`, `/api/b44/run-task`

### TTS (`tts/speak.py`)

Uses `edge-tts` (Microsoft Edge Neural TTS, no local GPU needed). Default voice `zh-TW-HsiaoChenNeural` (female). Three TW voices: HsiaoChenNeural / HsiaoYuNeural (female) / YunJheNeural (male). Auto-plays via mpv/ffplay/aplay after synthesis. Output saved to `~/桌面/tts_output.mp3`.

### Vision (`vision/`)

Two-stage pipeline: **moondream** (Ollama, English image recognition) → **qwen3.6** (Chinese translation + commentary). `describe_image.py` takes a file path; `ocpaste.py` additionally reads from clipboard (GTK/xclip/wl-paste fallback chain) or takes a screenshot (scrot/gnome-screenshot), then optionally sends the result to an OpenClaw agent session.

### Claude Code Bridge (`claude_bridge/`)

Wraps `claude --print` as an OpenAI-compatible REST API on port 19001. Two models: `claude-code` (uses `--dangerously-skip-permissions` for full tools) and `claude-code-no-tools` (uses `--tools ""`). Supports streaming (SSE) and non-streaming. 300s timeout. Intended for connecting Open-WebUI to Claude Code CLI.

Auto-starts via crontab `@reboot sleep 50`. **Known fix (2026-05-10):** `start.sh` must `cd "$SCRIPT_DIR"` before calling `uvicorn app:app`, otherwise uvicorn cannot find `app.py` and silently fails (PID is written but process immediately exits). If Open-WebUI loses the `claude.claude-code` models after reboot, check `~/ai_projects/logs/claude_bridge.log` for `Could not import module "app"` and verify `start.sh` still contains `cd "$SCRIPT_DIR"`.

### OpenClaw Proxy (`openclaw-proxy/`)

FastAPI server on port 11435. Exposes GPT-5.4/5.5 and Gemini 2.5 Flash through the OpenClaw gateway as OpenAI-compatible `/v1/chat/completions`. Supports streaming, image input (base64 → tempfile → `--file`), and logs to `~/ai_projects/logs/openclaw-proxy.log`.

### Health Monitoring

`monitor_health.sh` logs every 30 min via crontab. Monitors: CPU/RAM/Disk/GPU + Ollama + Dashboard + Open-WebUI + n8n + SearXNG. Alert thresholds: CPU>90%, RAM>85%, Disk>85%, GPU Temp>80°C, VRAM>95%.

## Key Files

- `~/ai_agent_env/` — primary venv for all agent scripts
- `~/marker_env/` — isolated venv for marker-pdf (avoids openai version conflict with crewai)
- `~/comfyui_env/` — isolated venv for ComfyUI
- `~/ai_projects/CLAUDE.md` — 模組總覽、執行指令、架構說明、環境變數
- `~/ai_projects/ppt/final_agent.py` — CrewAI + Pydantic 結構化 PPT（reference implementation）
- `~/ai_projects/excel/generate_excel.py` — CrewAI + openpyxl AI Excel 報表（--topic、--rows）
- `~/ai_projects/rag/ingest.py` — 文件嵌入 ChromaDB（marker-pdf 精準解析 PDF）
- `~/ai_projects/rag/query.py` — RAG 查詢 agent（CrewAI + kb_search tool）
- `~/ai_projects/whisper/transcribe_agent.py` — faster-whisper 語音轉文字（--summarize）
- `~/ai_projects/comfy/generate_image.py` — ComfyUI Stable Diffusion（--translate）
- `~/ai_projects/tts/speak.py` — edge-tts 繁體中文語音合成（--voice/--output/--list-voices）
- `~/ai_projects/tts_api/app.py` — TTS REST API（port 19002，WordPress 整合）
- `~/ai_projects/vision/describe_image.py` — moondream + qwen3.6 圖片分析（--speak）
- `~/ai_projects/vision/ocpaste.py` — 剪貼簿/截圖 → AI 分析 → OpenClaw
- `~/ai_projects/server_monitor/tools.py` — 所有 CrewAI 工具（6 tools：disk/health/local/search/whisper/comfy）
- `~/ai_projects/claude_bridge/app.py` — Claude Code CLI → OpenAI API bridge（port 19001）
- `~/ai_projects/openclaw-proxy/server.py` — OpenClaw → OpenAI proxy（port 11435，GPT/Gemini）
- `~/ai_projects/monitor_health.sh` — 健康監控腳本（crontab 每 30 分鐘，閾值警示）
- `~/ai_projects/preload_models.sh` — @reboot 預載 hermes3:3b + qwen3.6:latest
- `~/hermes-dashboard/main.py` — FastAPI web dashboard (port 7860)
- `~/hermes-dashboard/CLAUDE.md` — API 端點、頁籤說明、依賴與注意事項
- `~/.hermes_token` — dashboard API token (auto-generated, required for /api/b44/* endpoints)
- `~/ComfyUI/models/checkpoints/v1-5-pruned-emaonly.safetensors` — SD 1.5 model (4.27GB)
- `~/bin/start-all` — 啟動所有 AI 服務（Docker 容器 + Hermes Dashboard + ComfyUI + Claude Bridge）
- `~/bin/hm` — Hermes 模型快速切換（本地 Ollama + 雲端 Gemini/GPT）
- `~/bin/b44-info` — Hermes Dashboard API Token 與連線資訊顯示工具
- `~/bin/ocpaste` — 剪貼簿/截圖 → AI 分析 → OpenClaw（包裝 vision/ocpaste.py）
- `~/bin/CLAUDE.md` — 所有腳本用法、hm 模型速查表
- `~/.openclaw/openclaw.json` — jwd OpenClaw config (v2026.5.7, port 18789, primary: hermes3:3b)
- `~/.openclaw/openclaw.json.bak.*` — 設定備份（更新前自動產生）
- `~/setup_openclaw_juser1.sh` — juser1 OpenClaw 安裝腳本（v2026.5.6, port 18790）
- `~/penpot/docker-compose.yml` — 設計協作平台設定（5 containers，port 9001）
- `~/penpot/index.html` — 修改版前端首頁（繁中標題），bind mount 至容器 `/var/www/app/index.html`
- `~/penpot/CLAUDE.md` — 容器清單、中文化設定、Volume 與升級說明
- `~/gitea/docker-compose.yml` — 設計協作倉庫設定（APP_NAME=設計協作倉庫、DEFAULT_LOCALE=zh-TW）
- `~/gitea/CLAUDE.md` — 環境變數、SSH 設定、升級指引
