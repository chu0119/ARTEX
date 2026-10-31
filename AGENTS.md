# AGENTS.md — AI 协作规范（Agent Harness 指引）

> 本文件给在本仓库工作的 AI 编码代理（Claude Code / ZCode / Codex 等）看。
> 目标：不读聊天记录也能正确构建、测试、发版，不踩已知的坑。
> 人读的文档见 [README.md](README.md) 与 [docs/](docs/)。

## 0. 项目一句话

ARTEX = AI 自主渗透测试平台（Go 单体后端 + Next.js 静态导出前端 + PostgreSQL 唯一存储）。
planner/worker 双角色多 Agent 自主推进，双图架构（资产图×探索图），三层拦截规则，全程证据留痕。
接手自已删除的原作者仓库（基线 v0.3.15），现为独立演进项目。**仅供授权测试**（README 免责条款）。

## 1. 常用命令

```bash
# 后端构建（两条都要能过）
go build ./...

# 前端：类型检查 + 静态导出（注意：必须在 web/ 目录下跑；NEXT_EXPORT=1 是 Git Bash 语法，
# 不要用 `npm run build:static`——Windows cmd 下该 script 里的环境变量语法会挂）
cd web && npx tsc --noEmit
cd web && NEXT_EXPORT=1 npx next build

# 全量测试：连真 PG（Docker），-p 1 不能去（跨包死锁），先给新鲜库（共享库跑多轮会撞
# 固定名 fixture 的唯一键，产生假失败）
docker run -d --name artex-audit-pg -e POSTGRES_PASSWORD=artextest -e POSTGRES_USER=artex \
  -e POSTGRES_DB=artex -p 15432:5432 postgres:16-alpine
docker exec artex-audit-pg psql -U artex -d postgres -c "DROP DATABASE IF EXISTS artex_ci; CREATE DATABASE artex_ci OWNER artex"
ARTEX_PG_DSN="postgres://artex:artextest@127.0.0.1:15432/artex_ci?sslmode=disable" go test -p 1 ./... -count=1

# 发版构建（本机交叉编译验证用；正式发版靠 CI）
ARTEX_SKIP_FRONTEND=1 ARTEX_SKIP_NPM_CI=1 ARTEX_COMPRESS=0 ARTEX_PACKAGE=0 \
  ARTEX_OUTPUT=/abs/path/artex ./build.sh --target linux/amd64
```

## 2. 测试基线（判定 FAIL 是否与你有关）

- **已知负载性 flaky**（单跑必过，全量偶发，勿当回归）：
  `TestChatMentionWorkerReceivesServerDetails`、`TestNotifyDigestSegmentsAndDefersRemainder`（共享库脏数据）、
  `TestChatUnavailableReasonDistinguishesStates`（Windows TempDir 清理竞态）。
- **判定法**：`go test ./server/ -run <name> -count=1` 单跑，过 = 环境问题。
- **Windows 特有**：`evidence` 目录 fsync 已做 GOOS 分支；不要引入依赖目录 fsync 的新代码。

## 3. 发版流程（tag → CI → Release）

1. `CHANGELOG.md` 补节（**编辑前确认锚点存在**——历史上 str.replace 锚点失配静默吞过整节）。
2. commit → push main → `git tag v0.3.x` → push tag → GitHub Actions 自动出 5 平台 zip + SHA256SUMS + Docker + Release。
3. CI 门禁与本地等价：`go build ./...`、`go test -p 1`（真 PG）、前端 tsc + `NEXT_EXPORT=1 next build`。
4. **提示词改动会随 `SeedPromptVersioned` 自动下发**已部署实例（用户手改过的不受影响）——改 worker/planner 默认模板不用写迁移。
5. **拦截规则新增走 seed 批次**（v1…v5 先例）：除 `seedBuiltinInterceptRulesBatch` 外**必须**配按名补种 + enabled 工具集合并（v4/v5 有真实升级路径缺陷教训：flag 早退会吞掉新规则）。
6. git push 对 github.com 常因本机代理抖动失败——`git -c http.proxy= -c https.proxy= push` 直连 + 重试循环（30-45s 间隔）+ `git ls-remote` 确认落地。

## 4. 关键坑（真实踩过的）

- **默认构建不含前端**：`go build ./cmd/artex` 出的是 stub（页面报「前端未内嵌」）。内嵌必须
  `-tags embedui` 且先 `cp -r web/out server/webui/dist`。别把无 tags 的现场编译版当正式版部署。
- **dev 版本号禁用一键更新**：`CompareVersions` 只认纯净三段式；`0.3.95-dev` 这类会被判不可比较。
  本机交叉编译验证时传 `ARTEX_BUILD_VERSION` 要注意。
- **next build 的 cwd 必须是 `web/`**——曾在 `web/src` 下误跑，产物 `.next` 被 `git add -A` 带进仓库。
- **短 sleep 等定时器的测试不许写**：`1ms 超时 + sleep 2ms 断言` 在 CI 高负载翻车（本地必过）。
  用有界轮询。
- **工具实现了必须注册**：`agent/tools_*.go` 里的领域工具要进
  `PlannerTools/WorkerTools/MainAgentTools` 三张清单（tools.go / tools_insert.go），
  还要考虑 `agent/toolcatalog.go` 的 BuiltinToolSeeds。listener（v0.3.87）和
  list_worker_traces（v0.3.89）都栽过「实现了没注册」。
- **静态导出不支持服务端 redirect()**：会被静默丢弃、路由导出成 404 页。用客户端
  `router.replace`（v0.3.88 根路由修复先例）。
- **纯读工具用 readTool/readExpTool**：writeTool 会丢 ReadOnly/Concurrent 标记，被调度串行化。
- **不要在对话/报告里写**：FOFA key、LLM API key、任何密码明文——凭据只进运行时配置。

## 5. 目录速览

```
cmd/            入口（artex / artex-hostd 特权助手）
agent/          多 Agent 层（planner/worker/mainagent 提示词与工具；三张工具清单在 tools.go/tools_insert.go）
db/             PostgreSQL 唯一存储（schema.sql 幂等迁移；拦截规则 seed 批次 v1-v5）
server/         HTTP 面与引擎调度（全部路由注册在 server.go:680-1000 一带）
intercept/      三层拦截（规则+LLM judge+ask 审批；judge prompt 在 prompt.go）
listener/       回连接收器；guard/        审批门（Attributions 已回流 planner 态势）
traffic/ vulnintel/ llmpool/ selfupdate/ report/（手写 OOXML，勿引第三方文档库） skills/
web/            Next.js 前端（src/app 路由；lib/api.ts 单一 API 封装）
docs/           设计文档 + 阶段报告（按日期命名的是阶段性快照，越新的越权威）
```

## 6. 文档权威顺序（同一话题以新为准）

`docs/深度优化方案-2026-10-31.md`（当前路线图）> `docs/产品全面体检-2026-10-31.md`（缺陷基线）>
`docs/模型路由策略.md`（模型选型）> `docs/红队能力建设-2026-10-30.md` > `docs/产品优化清单-2026-10-29.md`
（历史，35% 完成度快照）。部署与排障：`docs/部署手册.md`（含 LAN 源安装配方）。
接手背景：`docs/项目交接文档.md`。
