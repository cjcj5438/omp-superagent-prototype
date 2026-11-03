# omp-superagent-prototype

一个跑在 [omp](https://github.com/oh-my-pi) 上的 Superagent：监听 GitHub 与飞书的消息，经过策略准入与评估，把明确的请求变成工单，再派 omp worker 去执行，最后把结果回复到原线程。

借鉴 `dsh-superagent-prototype` 项目的思路（来源 → 触发 → 评估 → 工单 → 执行的主循环、策略/准入模型、管理面板），为 omp 重写了执行层与托管方式：每个会话是一个 `omp --mode rpc` 子进程，守护进程是单个 `dist/daemon.js`，可被任何宿主以子进程方式托管（参考实现：omp-resume 的 Superagent 集成）。

当前状态：研究原型，可独立构建、测试、运行。

## 架构

```text
GitHub / 飞书
    │ 轮询 + 事件消费
    ▼
来源 (sources/) ──► 触发 (Trigger) ──► 策略门 (policy gate)
                                              │ admissible?
                        ┌─────────────────────┴─────────────────────┐
                        ▼                                             ▼
                 评估 (Eval, 主会话)                              忽略 (decision: ignore/not_a_member/…)
                        │ accept
                        ▼
                 工单 (Ticket) ──► 执行器派 worker 会话 ──► gh/lark 干活
                        │
                        ▼
                 回复原线程 (ack + 结果)
```

- **仓库布局**：`src/` 守护进程与面板源码，`plugin/` 注入 omp 子进程的扩展，`prompts/` 会话提示词，`skills/` 会话技能，`tests/` 51 个测试文件（另有 `fixtures/` 目录），`docs/` 000–008 设计文档。
- **构建产物**：`dist/daemon.js`（守护进程，unix socket 上提供 HTTP API）、`dist/panel.js` + `dist/panel.d.ts`（可嵌入的管理面板，React 19）。
- **状态**：单个 SQLite（`S/superagent.sqlite`）+ 会话 JSONL + 工单目录，全部在 state 目录 `S` 下；staging 用 `proper-lockfile` 保证单实例。

## 快速开始

前置：`omp` CLI、Node 22 或 omp 自带 Bun、`gh` ≥ 2.48、git ≥ 2.32；飞书可选（`lark-cli`）。

```sh
npm install
npm run build        # dist/daemon.js + dist/panel.js
npm run typecheck
npm test             # 51 文件，约 1100 用例
```

宿主启动守护进程（以 omp-resume 为例，实际参数见 `docs/006-deployment.md`）：

```sh
BUN_BE_BUN=1 omp run dist/daemon.js \
  --state-dir S --socket S/daemon.sock --omp "$(command -v omp)"
```

然后配三样东西：GitHub 仓库 + 智能体 CLI、飞书机器人（可选）、策略名单（谁能发起工作）。注意空名单意味着**谁都不能发起工作**，第一条消息会被判 `not_a_member` 忽略——这是最常见的"配了没反应"原因。

## 文档

按顺序读：

| 文档 | 内容 |
| --- | --- |
| `docs/000-philosophy.md` | 设计哲学 |
| `docs/001-domain.md` | 领域模型：触发/线程/工单/评估 |
| `docs/002-authorization.md` | 授权：策略门、准入表、工单授权 |
| `docs/003-execution.md` | 执行：派工、worker 生命周期、恢复 |
| `docs/004-prompts.md` | 提示词体系 |
| `docs/005-panel.md` | 管理面板：六个 tab 与三问 |
| `docs/006-deployment.md` | 部署：账号、启动检查、排障 |
| `docs/007-architecture.md` | 架构：进程模型与目录布局 |
| `docs/008-cookbook.md` | 常见操作手册 |

先看 `006-deployment.md` 配账号，再看 `002-authorization.md` 理解"名单 + @"的准入规则。

## 约束（已知限制）

- 单 GitHub 仓库：一个 daemon 只监听一个 `owner/name`。
- 同一 GitHub agent 账号 + 仓库不要双开 daemon；同一飞书 app 不要两个消费者（见 `006-deployment.md`）。
- 智能体必须是 GitHub user 账号（App 读不到通知）；来源会跳过作者就是智能体账号的内容，自己 @ 自己无反应。
- 跨仓库只能只读/手工报 PR：PR 绑定与 CI 轮询只认主仓库。

## License

MIT，见 `LICENSE`。原型思路来源 `dsh-superagent-prototype` 的 MIT 声明保留在该文件中。
