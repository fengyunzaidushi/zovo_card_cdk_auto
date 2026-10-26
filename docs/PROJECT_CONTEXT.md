# Zovo CDK 项目上下文

## 核验基线

- 整理日期：2026-10-22（Asia/Shanghai）。
- 仓库位置：`/opt/zovo_card_cdk_auto`；分支：`master`；基线提交：`b28ea2f`。
- 核验范围：本地源代码、包元数据、Git 状态及部分文档；未验证运行中的服务、线上版本或卡台 API。
- 本文件是可更新的交接摘要。跨会话使用前应重新核对 HEAD 和工作区，不把此快照视为实时状态。

## 已确认的项目事实

- 项目实现 CDK 兑换与卡台 Open API 对接，包含兑换预检、结果查询、代理换码、账单查询、后台发码与订单管理等入口。依据：`backend/internal/server/http.go`、`frontend/src/router/index.ts`。这些入口的存在不代表本次已做端到端验证。
- 后端为 Go + Gin，模块名 `github.com/tuzi/cdk-recharge-system`；`backend/go.mod` 声明 Go `1.24.1`。
- 数据层使用 `database/sql` + `github.com/mattn/go-sqlite3`；默认数据库路径为 `../data/cdk_recharge.db`（相对于进程工作目录），可由 `DB_PATH` 覆盖。初始化及建表见 `backend/internal/db/db.go`。SQLite 驱动依赖 CGO，构建约束见 `deploy/README-release.md`。
- 前端为 Vue 3 + TypeScript + Vite，包含 Pinia、Element Plus、Tailwind CSS 等依赖，依据 `frontend/package.json`。
- 管理页面入口是 `/ops`，登录 `/ops/login`，安装向导 `/ops/setup`；管理 API 为 `/api/v1/admin/*`，公开兑换 API 为 `/api/v1/public/cdk/*`。依据前后端路由文件。
- `VERSION` 为 `1.4.30`，`frontend/package.json` 的版本为 `1.4.27`；两处目前不一致，不据此判断已部署版本。

## 文件导航

| 工作主题 | 首选入口 |
| --- | --- |
| 启动、路由与鉴权 | `backend/cmd/server/main.go`、`backend/internal/server/http.go`、`backend/internal/server/middleware.go` |
| 数据库、表结构 | `backend/internal/db/db.go` |
| CDK 业务与卡台调用 | `backend/internal/handler/cardplatform_cdk.go`、`backend/internal/cardplatform/client.go` |
| 代理补发、地区与选卡 | `backend/internal/handler/agent_exchange.go`、`backend/internal/handler/card_pref.go`、`backend/internal/handler/card_selection.go` |
| 安装与配置 | `backend/internal/handler/setup.go`、`backend/internal/handler/settings.go`、`backend/internal/config/config.go` |
| 回调与后台同步 | `backend/internal/handler/webhook.go`、`backend/internal/plansync/sync.go` |
| 页面与导航 | `frontend/src/router/index.ts`、`frontend/src/views/`、`frontend/src/layouts/AdminLayout.vue` |
| 发布构建与部署说明 | `deploy/README-release.md`、`scripts/build-release-bundle.sh`、`scripts/build-linux-backend.sh`、`Dockerfile` |
| 上游接口参考（按需阅读） | `docs/zovocard-cdk-zh.md`、`docs/zovocard-openapi-zh.md` |
| 地区与订阅专项说明 | `docs/cdk-list-region.md`、`docs/x-premium-subscriptions.md` |
| 独立浏览器自动化 | `automation/package.json`、`automation/README.md` |

## 当前任务与交接

- 用户目标：创建一个管理当前项目上下文的项目技能；用户明确要求保存在项目根目录的 `.codex/skills/` 下，不使用全局技能目录。
- 已创建技能 `zovo-project-context`，位于 `.codex/skills/zovo-project-context/`；本文件作为项目内的持久记录入口。技能随项目文件一起管理，原全局副本已移除。
- 设计选择：技能保存恢复/更新流程，项目文件保存实际状态；根据代码核验旧文档；分别记录事实、历史与未知项。这是本次实现选择，并非新增业务需求。
- 尚未提出新的业务开发任务。下一次可按用户需求恢复上下文、继续明确任务，或保存新的进度。
- 2026-10-27（Asia/Shanghai）用户明确要求提交当前改动，让工作区干净。本轮将项目技能、上下文文档及 `new_prompt.md`、`old_prompt.txt` 纳入同一个本地提交。
- 两份提示词已阅读：`new_prompt.md` 为多站共存环境下的部署方案，`old_prompt.txt` 为旧版部署模板。仅作为文档归档，本轮没有执行其中的部署指令，也没有推送远端。

## 验证记录与入口

2026-10-22（Asia/Shanghai）在仓库根目录执行了 `git status --short`、`git branch --show-current`、`git log -4 --format='%h %s'`，并阅读上述关键代码和元数据。已确认本文件所述分支、提交、版本与实现入口。本次没有执行应用测试、构建或部署。

技能格式校验命令（仓库根目录）：`python3 /root/.codex/skills/.system/skill-creator/scripts/quick_validate.py .codex/skills/zovo-project-context`。本文件引用的 29 个项目路径已检查存在。这些检查不代表应用功能验证。

以下是后续按需使用的验证入口，**本次未执行**：

| 目录 | 命令 | 范围 |
| --- | --- | --- |
| `backend/` | `go test ./...` | 后端 Go 测试；运行前核对相关测试的外部依赖 |
| `frontend/` | `npm run build` | Vite 构建，不等于独立 TypeScript 类型检查或端到端验证 |
| `frontend/` | `npm run test:grace-recovery` | 已定义的 grace-recovery 专项脚本 |

`frontend/package.json` 中 `lint` 带有 `--fix`，会修改文件，不应视为纯只读检查。

## 历史差异与待核验

- `README.md`、`QUICKSTART.md` 中有 PostgreSQL/Ent 等旧说明；实际初始化代码使用 SQLite。`backend/internal/config/config.go` 仍保留 PostgreSQL 风格字段，不能据此推断数据库实现。
- `IMPLEMENTATION_SUMMARY.md` 的无认证描述与当前管理员 JWT 路由不一致；带日期的进度文档也存在同文内新旧状态混杂。仅作为历史背景，继续工作时以相关代码及实际验证为准。
- 最近提交记录涉及代理补发保留原付款地区、`pro_5x_renew` 套餐识别；这里只记录 Git 历史摘要，未核验这些功能的行为或测试结果。
- 尚未确认当前服务健康、线上版本、上游接口可用性及既往进度文档中“已上线”的现状；需要相关任务时再验证。
- 不将历史待办自动升级为当前需求；不在上下文中保存真实密钥、session/cookie、完整卡密、卡片信息或用户数据。
