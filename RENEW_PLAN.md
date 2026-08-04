# supervisord Renew 计划

> 本文档是本项目 **renew 工作流**的权威计划：目标、决策、分阶段任务与验收标准。
> 所有开发在 `renew` 分支上进行，合并回 `master` 前需逐个 Phase 验收。
> 状态图例：⬜ 未开始 · 🔄 进行中 · ✅ 已完成 · ⛔ 阻塞

---

## 1. 项目目标（愿景）

对 [ochinchina/supervisord](https://github.com/ochinchina/supervisord)（Python supervisord 的 Go 重实现）
进行一次现代化改造，使项目：

1. **现代化工程基础**：单一 Go module、依赖全部升级到最新稳定版、CI 使用现代 Go 版本、`go vet` 零告警。
2. **配置格式升级**：从 INI 平滑切换到 TOML（TOML 优先、INI 自动检测兼容），老用户配置零迁移成本。
3. **AI 能力**：提供符合 Claude Code Skills 标准（SKILL.md）的技能文件，让 AI agent 能直接操作
   进程管理、日志分析、配置诊断；后续规划 MCP（Model Context Protocol）集成。
4. **Windows 平台健壮性**：审计并修复已知 Windows 平台问题（daemon、信号、僵尸进程回收、
   pid 追踪、路径处理等），对齐 GitHub 已报告的 issues。
5. **保持兼容性**：配置格式、XMLRPC 接口、事件协议与 Python supervisor 生态的兼容性不被破坏。

---

## 2. 决策记录（ADR）

| # | 决策 | 结论 | 理由 |
|---|---|---|---|
| ADR-001 | 模块结构 | **合并为单 module**（删除 10 个子包独立 go.mod） | 依赖升级、测试、开发复杂度大幅降低，符合 Go 主流实践；子包保留为内部 package 目录 |
| ADR-002 | TOML 迁移 | **TOML 优先 + INI 兼容**（按扩展名/内容自动检测） | 现有 Python supervisord 用户依赖 INI，直接切换会破坏所有存量配置 |
| ADR-003 | AI 功能形态 | **Claude Code Skills（SKILL.md）**，后续规划 MCP | 让主流 AI agent 可直接复用项目技能，无需自研协议 |
| ADR-004 | Windows 修复 | **先审计再修**，结合 GitHub 已知 issues | 避免盲修，问题清单与修复方案需先落地文档 |
| ADR-005 | 分支策略 | 全部 renew 工作在 `renew` 分支进行 | 与 master 隔离，逐 Phase 验收后合并 |

---

## 3. 任务分阶段

### Phase 0：合并为单模块 🔄

**目标**：消除多模块结构，为后续所有工作打基础。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 0.1 | 删除子包独立 go.mod/go.sum（config/process/events/faults/logger/signals/types/util/xmlrpcclient） | ✅ | 子包目录下无 go.mod |
| 0.2 | 清理根 go.mod：移除子包 require 与 replace 块 | ✅ | `go mod tidy` 通过 |
| 0.3 | 修复多模块遗留 bug：`process_manager_test.go` 错误 import `supervisord/config` | ✅ | 正确导入 `github.com/ochinchina/supervisord/config` |
| 0.4 | 修复 `ctl.go` tailLog 不可达代码（vet 报 unreachable code） | ✅ | `go vet ./...` 零告警 |
| 0.5 | 修复 `process.go` 非格式化字符串传给 Errorf（vet 报 non-constant format string） | ✅ | `go vet ./...` 零告警 |
| 0.6 | 全量构建 + 测试通过 | ✅ | `go build ./...`、`go test ./...` 全绿 |

### Phase 1：依赖升级 ⬜

**目标**：所有直接/间接依赖升级到最新稳定版，替换废弃库。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 1.1 | 升级直接依赖：kardianos/service → v1.3.0、logrus → v1.9.4、prometheus/client_golang → v1.24.1 | ⬜ | go.mod 更新，构建通过 |
| 1.2 | 升级间接依赖：x/sys、x/net、protobuf、prometheus/common 等 | ⬜ | `go mod tidy` 后无遗留旧版本 |
| 1.3 | 评估废弃/停滞库：mitchellh/go-ps（2018）、ochinchina/go-reaper（2018）、ochinchina/gorilla-xmlrpc（2017）、robfig/cron、rogpeppe/go-charset | ⬜ | 给出"升级/替换/内嵌 fork"决策并执行 |
| 1.4 | 升级 CI：Go 1.18 → 最新稳定版，actions/setup-go v3 → v5 | ⬜ | CI 在 ubuntu 上全绿 |
| 1.5 | 全量测试 + 跨平台编译验证（linux/darwin/windows × amd64/arm64） | ⬜ | `go build` 各平台交叉编译通过 |

### Phase 2：Windows 审计与修复 ⬜

**目标**：审计 8 个 `*_windows.go` 及通用代码的平台差异，结合 GitHub issues 修复。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 2.1 | 拉取 GitHub 上 windows 相关 open/closed issues 清单 | ✅ | 清单已落地到 `docs/windows-issues.md` |
| 2.2 | 审计 daemonize_windows.go（守护进程化在 Windows 的行为） | ⬜ | 问题清单 + 修复 |
| 2.3 | 审计 signals/signal_windows.go（信号映射：CTRL_C/CTRL_BREAK 等） | ⬜ | 问题清单 + 修复 |
| 2.4 | 审计 zombie_reaper_windows.go（僵尸进程回收在 Windows 为空实现） | ⬜ | 问题清单 + 修复 |
| 2.5 | 审计 process/pdeathsig_windows.go、set_user_id_windows.go（子进程生命周期） | ⬜ | 问题清单 + 修复 |
| 2.6 | 审计 logger/log_windows.go、rlimit_windows.go、pidproxy/signal_windows.go | ⬜ | 问题清单 + 修复 |
| 2.7 | Windows 实测验证（真实 Windows 环境或 CI windows runner） | ⬜ | Windows CI job 通过 |

### Phase 3：TOML 配置迁移 ⬜

**目标**：TOML 优先、INI 自动检测兼容。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 3.1 | 选型 TOML 库（BurntSushi/toml 或 pelletier/go-toml/v2）并落地决策 | ⬜ | ADR 记录 |
| 3.2 | config 包重构：抽象配置后端接口（ini/toml 双实现） | ⬜ | 单元测试覆盖两种格式 |
| 3.3 | 自动检测：按扩展名（.toml/.ini/.conf）+ 内容特征（`[program:` 等）识别格式 | ⬜ | 测试覆盖边界情况 |
| 3.4 | 新增 `supervisord config convert` 子命令（ini → toml） | ⬜ | 转换后行为等价（diff 测试） |
| 3.5 | 示例配置：新增 supervisor.toml 示例，保留 INI 示例 | ⬜ | 文档更新 |
| 3.6 | README/agents.md 文档更新 + 兼容性回归测试 | ⬜ | 全部既有测试通过 |

### Phase 4：AI Skills 与 MCP ⬜

**目标**：AI agent 可直接操作本工具。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 4.1 | 设计 skills 目录结构（`.claude/skills/`），编写 SKILL.md 骨架 | ⬜ | 符合 Claude Code Skills 规范 |
| 4.2 | Skill：进程管理（start/stop/restart/status 的自然语言操作指引） | ⬜ | agent 实测可完成全流程 |
| 4.3 | Skill：日志分析（日志定位、轮转日志检索、错误排查指引） | ⬜ | agent 实测可用 |
| 4.4 | Skill：配置诊断（INI/TOML 配置校验、常见错误修复指引） | ⬜ | agent 实测可用 |
| 4.5 | 规划 MCP server（进程状态/操作/事件订阅为 MCP tools） | ⬜ | MCP 设计文档落地 `docs/mcp.md` |
| 4.6 | （可选）MCP server 实现与测试 | ⬜ | mcp 客户端可连接调用 |

### Phase 5：CI 与文档现代化 ⬜

**目标**：工程设施与文档跟上时代。

| # | Task | 状态 | 验收标准 |
|---|---|---|---|
| 5.1 | CI 增加 Go 版本矩阵（latest + 上一版）、windows-latest runner | ⬜ | 多平台 CI 全绿 |
| 5.2 | 更新 .goreleaser.yml（GoReleaser 新版格式校验） | ⬜ | goreleaser check 通过 |
| 5.3 | 更新 README（新配置格式、AI 功能、Windows 支持状态） | ⬜ | 文档与实现一致 |
| 5.4 | agents.md 同步更新（单模块结构已变化） | ⬜ | 与代码一致 |
| 5.5 | 全量回归：所有 Phase 验收标准复跑 | ⬜ | 全部通过 |

---

## 4. 验收流程

1. 每个 Phase 完成 → 更新本文档状态（✅）→ 提交并 push `renew` 分支。
2. 全部 Phase 完成后，`renew` 合并回 `master`（可 squash 保留整洁历史）。
3. 合并前要求：`go build ./...`、`go vet ./...`、`go test ./...` 全绿，
   以及 Phase 1.5 / 2.7 的跨平台验证记录。

## 5. 相关文件

- `agents.md` — 面向 AI 编码助手的项目指南（随架构变化同步更新）
- `docs/windows-issues.md` — Windows 问题清单（Phase 2 产出）
- `docs/mcp.md` — MCP 设计文档（Phase 4 产出）
