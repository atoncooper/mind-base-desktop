# MindBase Desktop

[![CI](https://github.com/atoncooper/mind-base-desktop/actions/workflows/ci.yml/badge.svg)](https://github.com/atoncooper/mind-base-desktop/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> MindBase 的**完全本地桌面端**（Tauri 2 + React 19 + Rust）：数据只存在本机 SQLite，
> 不依赖任何后端服务，装好即可离线使用个人知识库与本地 Agent 工作台。

## 简介

MindBase Desktop 把 B 站收藏与本地文档变成可检索、可问答、可复习的个人知识库，并把「聊天 Agent、笔记、题目、思维导图/白板、科研工作台、简历/幻灯片、长任务自动化」全部做进一个桌面应用。它沿用 MindBase Web 端的 Agent harness / RAG 设计思路，但按单机形态重写：没有账号体系、没有多用户服务端，不需要部署任何容器或数据库。

核心取舍是**隐私与零运维**：文档、向量、笔记、会话、API Key、思维导图都保存在本机一个可迁移的数据目录里；唯一的出网行为是调用你自己配置的模型供应商（DashScope / OpenRouter / DeepSeek 等）、B 站数据接口与 GitHub 更新检查。API Key 只存在本地 SQLite 的 `api_keys` 表，前端只能看到掩码预览，日志不记录原始 Key。

与 Web 端（[atoncooper/MindBase](https://github.com/atoncooper/MindBase)）是**同源演进的两个产品**：Web 端面向服务器上的多用户/共享场景，本仓库是纯本地单机版本。**两端数据互不相通**——没有账号同步、没有云端同步；桌面端数据不会上传到 Web 端，Web 端数据也不会下载到桌面端。

## 核心功能

### 本地知识库与检索

- **B 站收藏 → 知识库**：扫码登录、同步收藏夹，按分 P 入库（AI 大纲 → 音频 → ASR → 规则切块 → 向量化），本地 SQLite 存储，换机/迁移只需搬数据目录。
- **混合检索**：BM25 + 向量余弦的 RRF 融合检索，可选 DashScope `gte-rerank` 精排；失败自动回退融合顺序。
- **多来源导入**：本地/云文档支持 PDF、Word、PPT、HTML、Markdown、TXT 等，图片文字走 OCR；内置 FFmpeg/ffprobe sidecar 处理音视频。
- **ASR 双模式**：云端 DashScope，或内置嵌入式 Python 的本地 faster-whisper（无需 API Key，模型权重首次使用时下载）。

### Agent 对话

- **多 Agent 编排**：本地 ReAct harness + `delegate_to_agent` 子代理（memory / note / code / search / academic 等），支持 plan、`ask_user` 结构化交互、会话级并发锁与熔断。
- **过程可见**：思考过程流式展示、工具步骤、产物卡片（简历/幻灯片等文件可一键打开）、后台任务状态、token 用量统计。
- **联网能力**：`web_search` / `web_crawl`（DeepSeek Search/Fetch 优先，回退 SearxNG → Bing → Baidu）+ 网页正文抓取。
- **本机执行**：文件读写限制在 Agent 目录；`run_command`（默认关闭，三套 shell）与 `run_sandboxed`（Windows AppContainer / Linux bubblewrap / macOS seatbelt，可断网）区分可信与不可信代码；另有结构化 `git` / `gh` 工具。

### 笔记、题目与画布

- **Markdown 笔记**：自动保存 + 乐观并发、修订快照与回滚、粘贴 B 站链接生成带时间戳的视频锚点。
- **题目练习**：基于已入库内容生成单选/多选/简答/论述题，本地批改与历史题集。
- **思维导图与白板**：simple-mind-map 导图 + Excalidraw 白板，本地持久化。

### 科研工作台

- **科研项目**：六阶段推进、计划、事件流；文献库（检索/导入/BibTeX 导出/引用核验）、稿件、数据集与实验记录。
- **AI 辅助**：AI 选刊（匹配度 + 理由）、AI 研究计划（分阶段目标/交付物/风险，可逐项执行回填）、文献引用检查。
- **写作与导入**：Google Docs 风格写作台（大纲/AI 续写/AI 润色/引用检查/导出），论文导入（PDF/DOCX/MD/TXT/TeX/HTML，自动拆章、Word 公式转 LaTeX、参考文献解析），原生 Word/PDF 版式预览优先。
- **简历 / 幻灯片**：生成后作为产物卡片打开，可直接编辑或导出。

### 自动化与更新

- **自动化长任务**：AI 启动训练/构建/批处理并立即返回，日志按任务/日期落盘；后台调度器定期产出有界上下文，支持查看、重试、终止，应用重启自动标记中断任务。
- **SSH 远程**：主机管理、xterm 交互终端、SFTP、本地端口转发、会话日志，跨平台凭据走系统钥匙串。
- **内置更新器**：通过 GitHub Releases 的 `latest.json` 检查并签名校验更新。

## 平台支持

| 平台 | 状态 | 产物 / 说明 |
|---|---|---|
| Windows x64 | ✅ 主要支持 | NSIS（currentUser，中/英）与 MSI；`run_sandboxed` 使用 AppContainer 隔离（文件仅沙箱目录 + 无网络/可选联网） |
| macOS Apple Silicon | 🧪 Beta | `.dmg` 安装包与自动更新包（`.app.tar.gz`）；Keychain / seatbelt 已实现，仍建议备份数据后再升级 |
| Linux x64 | 🧪 源码构建（未发布安装包） | 需要 WebKitGTK 等 Tauri 依赖与 bubblewrap；可按 `cross-platform.md` 自行构建 deb/rpm/AppImage |

## 快速开始

前置：**Node.js ≥ 22**、**Rust stable（≥1.88）**、**Python 3**，以及对应平台的 [Tauri 2 依赖](https://tauri.app/start/prerequisites/)（Windows 需 MSVC toolchain；Linux 需 WebKitGTK 等）。三平台差异见 [docs/cross-platform.md](docs/cross-platform.md)。

```bash
git clone https://github.com/atoncooper/mind-base-desktop.git
cd mind-base-desktop

npm ci

# FFmpeg / ffprobe sidecar：binaries/ 被 gitignore，首次运行/打包前必须准备
npm run fetch:ffmpeg:win    # Windows（PowerShell 脚本下载/复制）
npm run fetch:ffmpeg        # macOS / Linux（需系统已装 ffmpeg，或加 --source 指定目录）

npm run tauri dev           # 开发运行（Vite devUrl http://localhost:1420）
npm run tauri build         # 打包安装包（Windows NSIS/MSI、macOS dmg 等）
```

**数据与密钥都在本地**：默认数据目录为操作系统 App Data 下的应用目录，可在应用内「数据存储」迁移到任意文件夹（自带迁移/自愈）；笔记、向量、会话与模型 API Key 全部落在同一个 SQLite 数据库中。API Key 只在 Rust 层使用，UI 仅展示掩码预览，不写日志、不上传任何服务器。除模型供应商、B 站与 GitHub 更新检查外，应用不做外部网络请求。

## 自动更新

应用通过 Tauri updater 检查本仓库 Release 中的 `latest.json`：

- endpoint：`https://github.com/atoncooper/mind-base-desktop/releases/latest/download/latest.json`（已写入 `src-tauri/tauri.conf.json`）；
- 更新包用内置公钥做签名校验（`plugins.updater.pubkey`），Windows 以 passive 模式静默安装；
- **发布 tag 约定为 `v<version>`**，且版本必须与 `src-tauri/tauri.conf.json` / `package.json` / `Cargo.toml` 一致；不一致时 Release workflow 会直接失败；
- `latest.json` 由 Release workflow 自动生成/合并（`windows-x86_64`、`darwin-aarch64`、`darwin-x86_64` 条目）。

## 持续集成与发布

| Workflow | 触发 | 内容 |
|---|---|---|
| `ci.yml` | PR / push `main` | Windows + macOS 矩阵：`npm ci` → `npm run build` → Rust stable → `cargo check --all-targets --locked` → `cargo test --lib --locked`（CI 会创建占位 ffmpeg sidecar，不下载真实二进制） |
| `release.yml` | push `v*` tag | 校验 tag 与 `tauri.conf.json` 版本一致 → 拉取 FFmpeg sidecar（Windows 用仓库脚本，macOS 用 ffmpeg-static release）→ `npm run tauri build`（带签名）→ Windows 创建 Release 并上传 NSIS/MSI + `latest.json`；macOS 等 Release 出现后上传 dmg 与 `.app.tar.gz`/`.sig` 并合并 darwin 更新条目；Release 正文优先使用仓库根 `RELEASE_NOTES_<version>.md` |

仓库需要配置的 Actions secrets：

- `TAURI_SIGNING_PRIVATE_KEY`：updater 私钥（对应 `tauri.conf.json` 的 `pubkey`；缺失会导致 `createUpdaterArtifacts` 构建失败）；
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`：私钥口令（没有口令时留空仍建议显式设置）。

`GITHUB_TOKEN` 由 Actions 内置，无需额外配置。

## 文档

| 文档 | 内容 |
|---|---|
| [docs/overview.md](docs/overview.md) | 产品定位、功能清单、运行与打包、本仓库目录总览 |
| [docs/cross-platform.md](docs/cross-platform.md) | Windows / macOS / Linux 构建前置、FFmpeg sidecar、系统凭据、Python 运行时与平台待办 |
| [docs/rust-architecture.md](docs/rust-architecture.md) | Rust 分包架构、依赖方向、沙箱实现、测试分层、SSH 与联网搜索设计 |
| [docs/automation.md](docs/automation.md) | 自动化长任务的目录/数据库、工具面、报告节奏、中断恢复与 token 统计 |

## 与主项目的关系

- Web 端：[atoncooper/MindBase](https://github.com/atoncooper/MindBase) —— 服务端多用户版本（账号、共享、云盘、会员等），需要部署服务。
- 本仓库：MindBase 的本地单机版本，强调隐私、离线与零运维；功能与设计同源，但**数据与账号互不相通**，也不共享服务端数据库。
- 桌面端的数据目录、SQLite schema 与升级迁移完全在本地完成；覆盖安装会自动迁移本地数据。

## License

[MIT](LICENSE) © 2026 via007
