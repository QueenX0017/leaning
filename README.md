# CX学习通 · Chaoxing Sign Research Lab

超星学习通签到系统的**协议研究 / 工程实践 / 本地测试**项目。核心产出：

- **研究管线**：二维码解析 → 参数分析 → 请求构造 → 人工确认发送 → 响应分析，每一步可观测、可测试、可回放
- **monitor**：无人值守自动签到（状态机 + QR 收件箱双通道）
- **三栏 Web UI**：QR Scanner / Request Inspector / Response Inspector + 实时日志（SSE）+ Monitor 面板 + 手机上传页

> ⚠️ **用途声明**：本项目用于协议研究与工程学习。**无人值守自动签到违反学校考勤纪律与学习通平台条款，使用后果自负。** 开发与测试默认全部指向本地 mock-server；指向真实平台需双开关显式开启（见下）。

## 架构

```
┌────────────────────── Web UI（三栏开发者工具）──────────────────────┐
│  QR Scanner │ Request Inspector │ Response Inspector               │
│  底部：实时日志（SSE）/ Monitor 状态面板        #/upload 手机上传页   │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ REST + SSE
┌──────────────────────────────▼─────────────────────────────────────┐
│ apps/server（Fastify :3000）  parse/upload/analyze/send/requests/   │
│                              logs-stream/health/monitor-status     │
└──────┬───────────────────────────────┬──────────────────────────────┘
       │ 共享 packages/*（shared/logger/config/protocol/qr-parser/     │
       │              request-builder）                                │
┌──────▼───────────────────────────────▼──────────────────────────────┐
│ apps/monitor（核心，独立进程 :3010）                                 │
│  状态机：IDLE→POLLING→(WAITING_QR|SIGNING)→COOLDOWN                 │
│  轮询 activelist（±20% 抖动）· QR 收件箱 chokidar 监听 · 文本响应映射  │
└──────┬───────────────────────────────┬──────────────────────────────┘
       │ 目标选择（TARGET=mock|real）    │
┌──────▼───────────────┐      ┌─────────▼───────────────┐
│ apps/mock-server :4000│      │ 真实学习通（仅显式开启）   │
│ 同路径剧本化镜像 ×6 场景│      │ passport2/mobilelearn 等 │
└──────────────────────┘      └─────────────────────────┘
```

完整设计文档见 [`docs/`](docs/00-roadmap.md)：架构 / API 契约 / 数据模型 / 错误码 / 日志 / monitor 设计 / 安全配置。

## 目录结构

```
apps/       mock-server（本地模拟学习通）· server（Fastify 主 API）
            monitor（无人值守签到，核心）· web（React+MUI 三栏 UI）
packages/   shared（类型/错误码）· logger（脱敏+SSE 总线）· config（env+Zod）
            qr-parser（jsqr 解码）· protocol（URL/请求/响应分析+DES）· request-builder（发送引擎）
tests/      集成 + E2E 全流程（真实三组件进程内互联）
docs/       设计文档集（01–07）+ 开发路线（00）
docker/     Dockerfile ×2 + nginx.conf + docker-compose.yml
```

## 快速开始（本地）

前置要求：Node.js ≥ 20、pnpm ≥ 12。

```bash
pnpm install          # 安装依赖
pnpm build            # 构建全部 workspace 包（tsc）
pnpm test             # 全仓测试（154 个，全绿）

# 依次启动（研究模式需要 mock + server + web；monitor 可选）：
pnpm --filter mock-server dev    # 1. :4000 本地模拟学习通
pnpm --filter server dev         # 2. :3000 主 API
pnpm --filter web dev            # 3. :5173 三栏 UI（浏览器打开）
pnpm --filter monitor dev        # 4. :3010 无人值守签到（可选）
```

> 生产形态运行：`pnpm --filter <pkg> build` 后 `node apps/<pkg>/dist/index.js`（环境变量来自根目录 `.env`，见 `.env.example`）。

### 玩法 A：研究管线（Web UI 手动模式）

1. 打开 `http://localhost:5173`，左栏上传/粘贴二维码 → 解析出 URL 与参数（enc 高亮）
2. 点击「构造请求 →」预填中栏 Request Inspector，可编辑后「分析构造」（不发送）
3. 右栏核对脱敏预览 → 勾选确认 → 「发送请求」→ 响应分析（状态/头/JSON 树）
4. 底部「实时日志」看 SSE 流；「请求历史」可回放任何一次构造

### 玩法 B：monitor 无人值守（mock 演练）

1. 启动 mock + server + monitor（上表 1/2/4），默认 `TARGET=mock`
2. mock 默认剧本会持续发布一个 QR 签到任务；Monitor 面板（UI 底部）观察相位流转：`waiting_qr`
3. 把二维码照片拖入 `data/qr-inbox/`，或用手机打开 `http://<本机IP>:5173/#/upload` 拍照上传（同一收件箱目录）
4. 面板显示 `sign success`；照片归档到 `data/qr-inbox/.processed/`

> 剧本演练：mock 端点支持 `?scenario=<name>` 或请求头 `X-Mock-Scenario`（success/fail/expired/duplicate/server_error/frequent/no_activity…），`GET http://localhost:4000/mock/scenarios` 查看全部剧本。

### 玩法 C：真实平台（自行承担风险）

需**同时**满足双开关（`docs/07` 防呆设计）：

```bash
TARGET=real
ALLOW_REAL_TARGET=true     # 缺一不可，否则 monitor 拒绝启动 / 请求被 REAL_TARGET_DISABLED 拦截
CX_PHONE=<学习通手机号>
CX_PASSWORD=<密码>
SIGN_LOCATION=113.516288,34.817038,河南省郑州市某地   # 位置签到预设（| 分隔多组轮试）
```

- 首次验证建议先用**研究管线**（Web UI 手动确认）观察真实端点行为，再开 monitor
- 登录为应用端 `fanyalogin`（密码 DES-ECB + base64，`packages/protocol/crypto.ts`，已与 openssl 对拍）；真实端点见 `docs/02 §D`
- 轮询间隔下限 3000ms 硬 clamp，避免高频风控（`docs/07 §1`）

## Docker（compose 四服务）

> 注：Docker 文件按仓库最佳实践编写；本机无 Docker 环境，**构建尚未实机验证**，遇到问题请提 issue。

```bash
docker compose -f docker/docker-compose.yml up --build
# 打开 http://localhost:5173（web 容器 nginx 反代 /api → server）
```

- 四容器：mock / server / monitor / web（nginx 静态托管）；`DATA_DIR` 共享卷（server 转发的 monitor 状态文件与 QR 收件箱双通道都在卷上）
- 容器网络内 mock 经服务名 `mock:4000` 寻址（`MOCK_HOST` 配置项）
- 真实平台模式：根目录 `.env` 写双开关与账号（compose `env_file` 带入）

## 配置（.env）

完整清单与默认值见 [`.env.example`](.env.example)，校验在 `packages/config`（Zod，启动即校验、产物只读冻结）。关键项：

| 变量 | 默认 | 说明 |
|---|---|---|
| `TARGET` / `ALLOW_REAL_TARGET` | `mock` / `false` | 目标选择双开关（防呆） |
| `QR_WATCH_DIR` / `QR_WAIT_TIMEOUT_MS` | `./data/qr-inbox` / `600000` | QR 收件箱与等待上限 |
| `POLL_INTERVAL_MS` / `MAX_TASK_AGE_MS` | `5000` / `7200000` | 轮询间隔（下限 3000）/ 任务窗口 |
| `CREDENTIAL_TTL_MS` / `SIGN_RETRY_MAX` | `21600000` / `2` | 会话刷新周期 / 签到重试次数 |
| `MAIL_*` | 关闭 | SMTP 结果通知（预留） |

## 测试

```bash
pnpm test                     # 全仓 154 个测试：单测 + 组件 + 集成 + E2E
pnpm --filter tests typecheck # tests/ 类型检查
pnpm build && pnpm lint       # 构建与 lint
```

- 单测：各 package/app 内 `src/*.test.ts`（协议/解码/状态机/会话/检测/分发/收件箱…）
- 组件测试：web（jsdom + Testing Library：MonitorPanel / RequestPanel）
- 集成 + E2E：`tests/` 真实三组件进程内互联（全流程签到、剧本全覆盖、NOT_CONFIRMED 门禁）

## 安全边界（红线，详见 `docs/05` / `docs/07`）

- 凭据只经 `.env` 注入；`password` 永不入日志（删除）、`_d/vc3/uf` 保留前 2 位、`enc` 保留前 4 位、URL 中 enc/uid 手动掩码
- `.env`、`data/`（日志/状态/收件箱/请求历史）整体 gitignore；仓库只提交 `.env.example`
- 不伪造设备身份、不绕过验证码、不向平台高频请求、不帮助他人批量代签

## 路线图与文档

进度与阶段汇报见 [`docs/00-roadmap.md`](docs/00-roadmap.md)；设计文档：架构 `01` / API 契约 `02` / 数据模型 `03` / 错误码 `04` / 日志 `05` / monitor `06` / 安全配置 `07`；**面向使用者的完整操作指南见 [`docs/08-使用手册.md`](docs/08-使用手册.md)**。

## License

MIT（学习与研究用途；真实平台使用后果自负）。
