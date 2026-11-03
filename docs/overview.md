# mind-base-desktop — 桌面端（Tauri 2）

> 全本地运行的桌面版 MindBase：Windows / macOS / Linux（源码构建）。与 MindBase Web 端**无运行时依赖**——不连服务端，数据（文档、向量、笔记、会话、API Key、思维导图）全部落在本机一个可迁移目录的 SQLite 中；Agent/RAG 设计与 Web 端同源。

## 定位

| 维度 | 说明 |
|------|------|
| 技术栈 | Tauri 2（Rust shell）+ React 19 + TypeScript + Vite 8 |
| 版本 | 0.2.0（`src-tauri/tauri.conf.json` / `src-tauri/Cargo.toml` / `package.json`；三处已对齐） |
| 数据 | 本机 SQLite（Tauri IPC → Rust 层读写），无网络后端依赖 |
| 外部调用 | 仅 LLM Provider（DashScope/DeepSeek/OpenRouter）、B 站（WBI 签名收藏夹/AI 大纲）、Context7 文档搜索、GitHub 更新检查 |
| 打包 | Windows NSIS（currentUser，中/英）/ MSI；macOS dmg（Apple Silicon，Beta）；Linux 源码构建（未发布安装包） |

## 功能

- **Chat 工作台**：Agent harness（6 个 agent 含 delegate 机制）、实时 agent 状态栏、后台长任务（注入消息/停止/预算）、plan 工具（分步检查点 + 剩余计划修订）
- **知识库**：B 站收藏导入 → ASR → 切块 → DashScope embedding → SQLite 余弦暴力检索；混合检索（BM25 + 向量 RRF）+ 可选 DashScope gte-rerank 精排（有 DashScope key 即生效，`MIND_BASE_RERANK=off` 关闭；失败自动回退融合顺序）；文件入库（云 + 本地 RapidOCR）
- **知识导图 + 自由白板**（0.1.8）：simple-mind-map + Excalidraw，本地存储
- **学术研究 agent**（0.1.9）：写/审/改/理解论文四种模式；提示词文件体系（6 个 agent 的 prompt 全部落为数据目录可编辑 Markdown，21 个文件，删了自动恢复，改完下一条消息生效）；记忆架构（会话黑板共享池 + 全量归档检索 + 每 agent 私有长期记忆 + 防串扰）
- **其他**：Quiz 生成与历史题集、本地 Markdown 笔记（自动保存/修订/视频锚点）、简历/幻灯片生成 agent、技能管理器、全局出口代理（双地址 scheme 路由）、内嵌 Python 本地 whisper ASR、FFmpeg/ffprobe sidecar、实验性视觉读图（默认关）
- **Agent 能力（2026-10 补齐）**：
  - `ask_user` 结构化交互：clarify 选项 / confirm 确认 / form 表单，调用即结束本轮，用户作答作为下一条消息（与 Web 端同语义）；
  - 产物卡片：本轮写入 `exports/` 的文件（简历、幻灯片等）通过 `artifact` 事件推到消息内，可一键打开；
  - 后台任务：聊天顶栏「后台任务」把任务派给 6 个 agent（chat/memory/note/code/search/academic），Agent 状态页查看/注入/停止；
  - 联网搜索 `web_search`：SearxNG（设 `MIND_BASE_SEARXNG_URL` 优先走自建实例）→ Bing → Baidu 引擎链，先搜索再用 `web_crawl` 抓正文；
  - 主动图片输入：输入框支持粘贴/拖拽/选择图片（≤4 张、单张 ≤5MB、总计 ≤15MB），仅在视觉模型下启用；图片只随本轮请求发送，不落库（刷新后不重放）；
  - token 计量：每轮汇总全部 ReAct 轮次（含成功委托的子代理）的 input/output/total tokens，随消息落库（`chat_messages.usage`，旧库启动自愈补列）并在消息底部展示；流式请求默认带 `stream_options.include_usage`，可用 `MIND_BASE_STREAM_USAGE=off` 关闭；
  - 工具执行硬超时：每个工具独立线程执行，runtime 按 `MIND_BASE_TOOL_TIMEOUT_SECS`（默认 180s）等待；卡死工具降级为「工具执行超时」文本、本轮继续，其余工具结果不受影响（超时线程 detach，持有 owned Arc 数据）；
  - 文件工具沙箱：`read_file`/`write_file`/`list_dir` 相对路径固定在本 agent 目录 `agent-files/<agent>/`，绝对路径仅限应用数据目录内。

## 运行与打包

前置：Node ≥ 22、Rust stable（≥1.88）+ Tauri 2 依赖、Python 3。

```bash
npm ci
npm run fetch:ffmpeg:win    # Windows：下载/复制 ffmpeg + ffprobe sidecar
npm run fetch:ffmpeg        # macOS / Linux：需系统已装 ffmpeg，或 --source 指定目录

npm run tauri dev           # 开发（Vite devUrl http://localhost:1420）
npm run tauri build         # 打包（Windows NSIS/MSI、macOS dmg 等）
```

> ⚠️ 已知待办：FFmpeg sidecar 未做哈希/版本锁定（供应链 TODO，见本项目根 README）。

## 目录结构

```text
mind-base-desktop/                  # 独立仓库根 = 应用根
├── README.md / RELEASE_NOTES_*.md
├── LICENSE                         # MIT
├── package.json / package-lock.json / vite.config.ts / tsconfig*.json
├── index.html
├── docs/                           # overview / cross-platform / rust-architecture / automation
├── public/                         # 静态资源
├── scripts/                        # fetch-ffmpeg.ps1 / fetch-ffmpeg.mjs（sidecar 准备）
├── src/                            # React 19 前端（components/ + lib/ IPC 封装）
└── src-tauri/                      # Rust shell
    ├── tauri.conf.json / Cargo.toml / Cargo.lock / build.rs
    ├── src/                        # 业务模块（app/agent/knowledge/media/llm/workspace/infra/…）
    ├── prompts/                    # 各 agent 提示词（可编辑数据）
    ├── resources/ / capabilities/ / tests/
    └── binaries/                   # ffmpeg/ffprobe sidecar（gitignored）
```

## 发布

CI：PR / main 由 `.github/workflows/ci.yml` 跑前端构建 + `cargo check/test`（Windows + macOS）；
tag `v<version>`（与 `src-tauri/tauri.conf.json` 版本一致）触发 `.github/workflows/release.yml`——Tauri 签名构建（Windows NSIS/MSI + macOS dmg、`.app.tar.gz`）+ FFmpeg sidecar，上传安装包与 `latest.json` 到 GitHub Release。详见根 README。
