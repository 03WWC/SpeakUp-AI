# SpeakUp AI / OfferGPT — 后端 Python 结构说明

> 文档说明 `backend/` 目录下各 Python 文件的职责，以及主要 REST / WebSocket 调用链路。  
> 生成时间：2026-07-03

---

## 一、项目总览

```
qibiuyun/
├── backend/                 # FastAPI 后端（核心 Python 代码）
│   ├── main.py              # 应用入口
│   ├── config.py            # 配置
│   ├── database.py          # 数据库连接与初始化
│   ├── exceptions.py        # 统一异常
│   ├── auth/                # JWT 认证
│   ├── models/              # SQLAlchemy ORM
│   ├── routers/             # REST API 路由
│   ├── services/            # 业务逻辑
│   │   └── realtime/        # 实时语音分析 Agent
│   ├── websocket/           # WebSocket 实时对话
│   └── migrations/          # SQL 迁移（非 .py）
├── tests/backend/           # 后端单元/集成测试
├── scripts/
│   └── run_migration.py     # PostgreSQL 迁移脚本
└── frontend/                # Next.js 前端（TypeScript，非 Python）
```

---

## 二、每个 `.py` 文件说明

### 2.1 根层 / 基础设施

| 文件 | 作用 |
|------|------|
| `backend/main.py` | **应用入口**：创建 FastAPI、注册 CORS/异常处理、挂载 5 个 REST 路由、定义 WebSocket `/ws/interviews/{session_id}` 和 `/health`；启动时连接 Redis/内存缓存、初始化数据库、预加载 Whisper ASR |
| `backend/config.py` | **配置中心**：从 `.env` / `.env.local` 读取环境变量（端口、JWT、LLM、Redis、TOS 对象存储、VAD/TTS 参数等），提供 `settings` 单例 |
| `backend/database.py` | **数据库层**：创建 async SQLAlchemy 引擎/会话工厂；`get_db()` 供路由依赖注入；`init_db()` 建表、迁移 `hashed_password` 列、种子 demo 用户和演示会话数据 |
| `backend/exceptions.py` | **统一错误格式**：`ApiError` 业务异常 + `error_response()`；注册 FastAPI 全局 handler（业务错误 / HTTP 404 / 参数校验失败） |

---

### 2.2 `backend/auth/` — 认证模块

| 文件 | 作用 |
|------|------|
| `auth/__init__.py` | 模块导出：JWT、密码、Schema、`get_current_user` |
| `auth/jwt.py` | **JWT 签发/验证**：`create_access_token` / `create_refresh_token` / `decode_token` |
| `auth/passwords.py` | **密码哈希**：bcrypt 加密与校验 |
| `auth/schemas.py` | **Pydantic 模型**：注册/登录/刷新请求体、`TokenResponse`、`UserResponse` |
| `auth/dependencies.py` | **FastAPI 依赖**：`get_current_user` 从 `Authorization: Bearer` 解析用户；demo 模式无 token 时回落 demo 用户 |

**主要 URL：**

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/auth/register` | 注册，返回 JWT |
| POST | `/api/auth/login` | 登录 |
| POST | `/api/auth/refresh` | 刷新 token |
| GET | `/api/auth/me` | 当前用户信息 |

---

### 2.3 `backend/models/` — 数据模型

| 文件 | 作用 |
|------|------|
| `models/__init__.py` | 导出所有 ORM 类 |
| `models/base.py` | **SQLAlchemy ORM 定义** |

**ORM 表一览：**

| 模型 | 表名 | 说明 |
|------|------|------|
| `User` | `users` | 用户账号、plan、密码哈希 |
| `Resume` | `resumes` | 上传简历及解析画像 |
| `Job` | `jobs` | 岗位 JD 及解析画像 |
| `ScenePreset` | `scene_presets` | 用户场景预设 |
| `Interview` | `interviews` | 口语练习会话（transcript、metrics、状态） |
| `TimelineEvent` | `timeline_events` | 时间轴事件（STAR 缺失、语法问题等） |
| `Report` | `reports` | 课后报告（Offer 评分、维度分） |
| `AgentLog` | `agent_logs` | Agent 调用日志（LLM 延迟、token 等） |

---

### 2.4 `backend/routers/` — REST API 路由层

| 文件 | 作用 | 主要 URL |
|------|------|----------|
| `routers/__init__.py` | 空包标识 | — |
| `routers/auth.py` | 用户认证 | `/api/auth/*` |
| `routers/scenes.py` | 场景配置列表 | `GET /api/scenes` |
| `routers/resumes.py` | 简历上传解析 | `POST /api/resumes` |
| `routers/jobs.py` | JD 创建解析 | `POST /api/jobs` |
| `routers/interviews.py` | 会话全生命周期 | 见下表 |

**`interviews.py` 主要接口：**

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/interviews` | 创建会话，返回 WebSocket URL |
| GET | `/api/interviews` | 用户历史会话列表 |
| DELETE | `/api/interviews/{session_id}` | 删除会话 |
| GET | `/api/interviews/{session_id}` | 会话详情 |
| POST | `/api/interviews/{session_id}/finish` | 结束会话，触发报告生成 |
| GET | `/api/interviews/{session_id}/analysis` | 发音/语法/语气词分析 |
| GET | `/api/interviews/{session_id}/replay/full` | 整场录音回放 |
| GET | `/api/interviews/{session_id}/replay/{turn_id}` | 单轮录音回放 |
| GET | `/api/interviews/{session_id}/events` | 时间轴事件 |
| GET | `/api/interviews/{session_id}/report` | 课后报告 |
| POST | `/api/asr/switch` | 切换 ASR 模型 |
| GET | `/api/asr/status` | ASR 状态 |
| GET | `/api/demo` | Demo 演示数据 |

---

### 2.5 `backend/services/` — 业务服务层

| 文件 | 作用 |
|------|------|
| `services/__init__.py` | 导出 `scene_service`、`cache` |
| `services/scene_service.py` | **场景配置服务**：静态 `SCENES_CONFIG`（面试/餐厅/会议等），提供 `get_scene_list` / `get_all_scenes` / `get_scene` |
| `services/cache_service.py` | **缓存服务**：优先 Redis，本地开发自动降级内存缓存；会话状态、分析数据 TTL 存储 |
| `services/llm_service.py` | **LLM 结构化提取**：调用 DeepSeek API 解析简历/JD JSON；无 API Key 时用正则兜底 |
| `services/resume_service.py` | **简历处理**：PDF/TXT 文本抽取、本地文件保存、调用 `llm_service.parse_resume_profile` |
| `services/job_service.py` | **JD 处理**：校验输入，调用 `llm_service.parse_job_profile` |
| `services/asr_service.py` | **语音识别**：本地 Whisper 转录 + `EnergyVAD` 能量检测；base64 PCM 解码 |
| `services/tts_service.py` | **语音合成**：EdgeTTS 在线合成或 Mock 空音频；按句分片流式输出 |
| `services/conversation_service.py` | **对话引擎**：按场景构建 System Prompt，调用 DeepSeek 流式对话，管理最近 6 轮历史 |
| `services/storage_service.py` | **对象存储**：优先火山 TOS，否则本地 `storage/`；上传 turn/整场 WAV 音频 |
| `services/session_persist_service.py` | **会话持久化**：把 cache 中的 transcript、分析数据、音频 URL 刷入 `Interview` 表；构建 analysis API 响应 |
| `services/report_service.py` | **规则报告引擎**：从 `analysis_store` 汇总发音/语法/语气词，规则计算 Offer 评分（不依赖 LLM） |
| `services/report_agent.py` | **LLM 报告 Agent**：用 DeepSeek 生成带证据的中文报告和时间轴；失败返回 None 由规则引擎兜底 |

---

### 2.6 `backend/services/realtime/` — 实时分析

| 文件 | 作用 |
|------|------|
| `realtime/__init__.py` | 导出 `grammar_agent`、`pronunciation_agent`、`analysis_store` |
| `realtime/asr_filter.py` | **ASR 过滤器**：置信度/词数/重复/非英文等多层校验，防误唤醒和噪音 |
| `realtime/grammar_agent.py` | **语法 Agent**：规则 + LLM 检测严重语法错误；统计语气词；触发 `correction.light` 或写入 cache |
| `realtime/pronunciation_agent.py` | **发音 Agent**：从 PCM 计算 WPM、停顿次数、低置信度词 |
| `realtime/analysis_store.py` | **分析数据存储**：基于 cache 存 `corrections` / `fillerCounts` / `pronunciation`，供报告使用 |

**Cache Key 约定：**

| Key | 内容 |
|-----|------|
| `session:{session_id}` | 会话 transcript、对话 history、整场音频 URL |
| `analysis:{session_id}` | 语法纠正、语气词计数、发音分析 |

---

### 2.7 `backend/websocket/` — 实时对话

| 文件 | 作用 |
|------|------|
| `websocket/handler.py` | **WebSocket 核心**：`WSManager` 管理连接与会话状态；路由 `audio.input` / `text.input` / `control.finish` 等消息；串联 VAD → ASR → 过滤 → LLM → TTS → 异步分析 |

**WebSocket 端点：** `ws://{host}/ws/interviews/{session_id}?token={accessToken}`

**主要消息类型：**

| 类型 | 方向 | 说明 |
|------|------|------|
| `audio.input` | 客户端 → 服务端 | 推送 PCM 音频帧 |
| `audio.turn.end` | 客户端 → 服务端 | 客户端 VAD 检测到静音，触发 ASR |
| `text.input` | 客户端 → 服务端 | 文本输入（Demo 主路径） |
| `control.finish` | 客户端 → 服务端 | 结束对话 |
| `control.correction` | 客户端 → 服务端 | 开关实时轻纠正 |
| `control.listening` | 客户端 → 服务端 | TTS 播完，恢复录音 |
| `ping` / `pong` | 双向 | 心跳 |
| `asr.final` | 服务端 → 客户端 | ASR 最终识别结果 |
| `ai.text.delta` | 服务端 → 客户端 | LLM 流式文本 |
| `ai.audio.delta` | 服务端 → 客户端 | TTS 音频片段 |
| `correction.light` | 服务端 → 客户端 | 实时语法轻纠正 |
| `turn.phase` | 服务端 → 客户端 | 对话阶段（listening / ai_speaking 等） |

---

### 2.8 `scripts/` 与 `tests/`

| 文件 | 作用 |
|------|------|
| `scripts/run_migration.py` | 读取 `.env.local` 的 `DATABASE_URL`，执行 `001_init.sql`，校验 8 张核心表 |
| `tests/backend/conftest.py` | pytest 夹具：测试客户端、数据库、mock 配置 |
| `tests/backend/test_exceptions.py` | 异常处理测试 |
| `tests/backend/test_cache_service.py` | 缓存服务测试 |
| `tests/backend/test_llm_service.py` | LLM 服务测试 |
| `tests/backend/test_scene_service.py` | 场景服务测试 |
| `tests/backend/test_job_service.py` | JD 解析测试 |
| `tests/backend/test_resume_service.py` | 简历解析测试 |
| `tests/backend/test_report_service.py` | 规则报告测试 |
| `tests/backend/test_asr_filter.py` | ASR 过滤器测试 |
| `tests/backend/test_grammar_agent.py` | 语法 Agent 测试 |
| `tests/backend/test_pronunciation_agent.py` | 发音 Agent 测试 |
| `tests/backend/test_websocket_handler.py` | WebSocket 处理器测试 |
| `tests/backend/test_router_scenes.py` | 场景路由测试 |
| `tests/backend/test_router_jobs.py` | JD 路由测试 |
| `tests/backend/test_router_resumes.py` | 简历路由测试 |
| `tests/backend/test_router_interviews.py` | 会话路由测试 |
| `tests/backend/test_integration_flow.py` | 端到端集成测试 |

---

## 三、分层架构

```
┌─────────────────────────────────────────────────────────────┐
│  前端 Next.js（frontend/）                                    │
│  REST: /api/*    WebSocket: /ws/interviews/{id}              │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  入口层  main.py                                               │
│  - FastAPI app / lifespan / CORS / 异常处理                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐
│  routers/     │  │  websocket/   │  │  /health              │
│  REST 路由     │  │  handler.py   │  │                       │
└───────┬───────┘  └───────┬───────┘  └───────────────────────┘
        │                  │
        └────────┬─────────┘
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  services/  业务逻辑层                                         │
│  conversation / asr / tts / llm / report / storage / cache   │
│  └── realtime/  grammar_agent / pronunciation_agent / filter │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  models/      │  │  database.py  │  │  cache        │
│  ORM 定义      │  │  PostgreSQL/  │  │  Redis/内存    │
│               │  │  SQLite       │  │               │
└───────────────┘  └───────────────┘  └───────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  基础设施  config.py / exceptions.py / auth/                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、核心调用链路

### 4.1 用户注册 / 登录

```
POST /api/auth/register
  → routers/auth.py :: register()
    → database.get_db() → models.User
    → auth/passwords.hash_password()
    → auth/jwt.create_access_token() + create_refresh_token()
    → 返回 TokenResponse

POST /api/auth/login
  → routers/auth.py :: login()
    → 查 User → auth/passwords.verify_password()
    → 签发 JWT → TokenResponse

其他需登录接口
  → Depends(get_current_user)  [auth/dependencies.py]
    → auth/jwt.decode_token()
    → db.get(User)
```

---

### 4.2 创建口语练习会话

```
POST /api/interviews
  → routers/interviews.py :: create_session()
    → get_current_user()
    → services/scene_service.get_scene()     # 校验场景/话题
    → 查 Resume / Job（可选）
    → 写 Interview 表
    → cache.set("session:{id}")              # 会话元数据
    → 返回 sessionId + websocketUrl
```

---

### 4.3 实时语音对话（WebSocket 主链路）

```
ws://host/ws/interviews/{sessionId}?token=...
  → main.py :: websocket_endpoint()
    → auth/jwt.decode_token()（非 demo 模式）
    → websocket/handler.py :: WSManager.connect()
    → WSManager.handle_message()
```

**音频路径 `audio.input`：**

```
handle_message(type=audio.input)
  → _handle_audio_input()
    → asr_service.decode_base64_pcm()
    → EnergyVAD 检测静音 → 累积 buffer
    → _flush_audio_turn() → _process_audio_turn()
        ├─ storage_service.upload_turn_audio()     [并行]
        ├─ asr_service.transcribe()              [Whisper]
        ├─ asr_filter.check()                    [过滤]
        ├─ _run_conversation_pipeline()
        │    ├─ conversation_service.stream_reply()  [DeepSeek 流式]
        │    └─ tts_service.synthesize_stream()      [EdgeTTS]
        └─ _run_async_analysis()                 [后台 task]
             ├─ grammar_agent.analyze()
             │    └─ analysis_store.append_correction()
             └─ pronunciation_agent.analyze()
                  └─ analysis_store.append_pronunciation()
```

**文本路径 `text.input`（Demo 常用）：**

```
_handle_text_input()
  → asr_filter.check()
  → _run_conversation_pipeline()    # 跳过 ASR，直接 LLM+TTS
  → _run_async_analysis()           # 仅语法，无发音
```

---

### 4.4 结束会话 & 生成报告

```
POST /api/interviews/{id}/finish
  → interviews.py :: finish_session()
    → session_persist_service.flush_session_data()
         ├─ analysis_store.load()
         ├─ cache.get("session:{id}")  # transcript + 音频 URL
         └─ 写入 Interview.transcript / metrics_json
    → asyncio.create_task(_generate_report_background())
         ├─ analysis_store.get_summary()
         ├─ report_agent.generate()        [LLM 优先]
         │    └─ 写 Report + TimelineEvent
         └─ report_service.build_report_payload()  [LLM 失败时规则兜底]

GET /api/interviews/{id}/report
  → 读 Report 表 → 返回评分/维度/建议

GET /api/interviews/{id}/analysis
  → session_persist_service.build_analysis_response()
  → enrich_analysis_from_timeline()
```

---

### 4.5 简历 / JD 上传

```
POST /api/resumes
  → resumes.py :: upload_resume()
    → resume_service.detect_file_type()
    → resume_service.extract_text_from_file()   [pypdf / txt]
    → resume_service.parse_resume()
         └─ llm_service.parse_resume_profile()
    → resume_service.save_resume_file()
    → 写 Resume 表

POST /api/jobs
  → jobs.py :: create_job()
    → job_service.parse_job()
         └─ llm_service.parse_job_profile()
    → 写 Job 表
```

---

### 4.6 应用启动

```
uvicorn main:app
  → lifespan 启动
    → cache_service.connect()        # Redis 或内存
    → database.init_db()             # 建表 + demo 数据
    → asr_service.preload()          # 预加载 Whisper（非 mock 模式）
  → include_router × 5
  → 注册 exception_handler × 3
```

---

## 五、完整练习流程（时序）

```
用户/前端          interviews 路由        WebSocket handler       服务层              存储
    │                    │                      │                   │                  │
    │ POST /interviews   │                      │                   │                  │
    │───────────────────>│ 写 Interview+cache   │                   │                  │
    │<── sessionId+wsUrl │                      │                   │                  │
    │                    │                      │                   │                  │
    │ WS 连接 + audio/text                      │                   │                  │
    │──────────────────────────────────────────>│ ASR→LLM→TTS      │                  │
    │                    │                      │──────────────────>│ analysis_store   │
    │                    │                      │                   │─────────────────>│ cache
    │                    │                      │                   │                  │
    │ POST /finish       │                      │                   │                  │
    │───────────────────>│ flush_session_data   │                   │                  │
    │                    │─────────────────────────────────────────────────────────────>│ DB
    │                    │ 后台 report_agent    │                   │                  │
    │                    │─────────────────────────────────────────────────────────────>│ Report
    │ GET /report        │                      │                   │                  │
    │───────────────────>│                      │                   │                  │
    │<── Offer 评分      │                      │                   │                  │
```

---

## 六、快速定位指南

| 想改什么 | 看哪个文件 |
|----------|------------|
| 新增 REST 接口 | `routers/` + `main.py` 注册 |
| 登录 / JWT | `auth/` |
| 数据库表结构 | `models/base.py` + `database.py` |
| 场景 / 话题配置 | `services/scene_service.py` |
| 实时对话逻辑 | `websocket/handler.py` |
| 语音识别 | `services/asr_service.py` |
| AI 对话 prompt | `services/conversation_service.py` |
| 实时纠错 | `services/realtime/grammar_agent.py` |
| 课后报告 | `services/report_agent.py` + `report_service.py` |
| 音频 / 文件存储 | `services/storage_service.py` |
| 环境变量 | `config.py` + 根目录 `.env.local` |

---

## 七、相关文档

- [前后端代码结构速查](./前后端代码结构速查.md)
- [代码说明文档](./代码说明文档.md)
- [运行操作手册](./运行操作手册.md)
- [API 契约](./agent-team/api-contract.md)
