# agents.md — 项目指南（面向 AI 编码助手）

> 本文件帮助 AI 代理快速理解 `supervisord` 代码库，避免在修改代码前做大量摸索。

## 项目是什么

这是 [supervisord](https://github.com/ochinchina/supervisord) 的 **Go 语言重实现版**。
Python 版 supervisord 需要目标环境安装 Python 解释器，在 Docker 等场景下过于臃肿；
本项目用 Go 重写，编译产物为单一静态二进制，适合无 Python 环境。

- 语言/版本：Go 1.24（见 `go.mod`，module 为 `github.com/ochinchina/supervisord`）
- 日志库：`github.com/sirupsen/logrus`
- 路由：`gorilla/mux`、XMLRPC 用 `gorilla/rpc` + `ochinchina/gorilla-xmlrpc`
- 配置解析：`ochinchina/go-ini`（兼容 Python supervisor 的 INI 格式）
- 监控指标：`prometheus/client_golang`（通过 HTTP 暴露 `/metrics`）

## 快速上手（常用命令）

```bash
# 编译（linux 静态二进制，需先 go generate 嵌入 webgui 资源）
go generate
GOOS=linux go build -tags release -a -ldflags "-linkmode external -extldflags -static" -o supervisord

# Windows 编译
go build -tags release -o supervisord.exe

# 运行
supervisord -c /path/to/supervisor.conf
supervisorctl -s http://127.0.0.1:9001 status

# 测试（各子包有独立 go.mod，需在对应目录执行）
go test ./...            # 根目录（主包）
cd process && go test ./...
cd config  && go test ./...
```

## 架构总览

```
┌────────────────────────── 主程序（根目录 package main）──────────────────────────┐
│  main.go          入口：命令行参数(-c/-d/--env-file)、信号处理、加载环境文件        │
│  supervisor.go    Supervisor 核心：持有 config/process.Manager/XMLRPC/logger       │
│  xmlrpc.go        XMLRPC 服务（HTTP + BasicAuth），暴露 supervisord 管理接口       │
│  ctl.go           supervisorctl 客户端命令（status/start/stop 等）                 │
│  webgui.go        Web 界面入口（嵌入 webgui/ 静态资源，assets_*.go 是 go:embed）   │
│  rest-rpc.go      REST 风格 RPC 接口                                              │
│  confApi.go       运行时配置查询接口                                              │
│  service.go       系统服务安装（daemon）                                           │
│  daemonize.go     守护进程化                                                      │
│  content_checker.go  配置文件内容变更检测（热重载）                                │
│  logtail.go       日志尾部跟踪                                                     │
│  rlimit*.go       Linux rlimit 设置（按平台分文件）                                │
│  zombie_reaper*.go  回收僵尸进程（按平台分文件）                                   │
│  version.go / config_template.go / supervisor.ini                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## 子包职责

| 包 | 职责 | 关键文件 |
|---|---|---|
| `config/` | 解析 supervisor.conf（INI 格式）、程序组管理、字符串表达式 | `config.go`、`process_group.go`、`string_expression.go`、`process_sort.go` |
| `process/` | **进程管理核心**：进程模型、启动/停止/重启、状态机、命令解析、指标 | `process.go`、`process_manager.go`、`command_parser.go`、`proc_metrics.go`、`process_change_monitor.go` |
| `events/` | 事件监听协议（Python supervisor 兼容的 EVENT 协议） | `events.go` |
| `faults/` | XMLRPC 错误码（FAULTS 定义） | `faults.go` |
| `logger/` | 进程日志管理：文件日志、按大小/时间轮转、stdout/stderr 分流 | `log.go`、`log_unix.go`、`log_windows.go` |
| `signals/` | 平台相关信号名与信号值映射 | `signal.go`、`signal_darwin.go`、`signal_windows.go` |
| `types/` | 公共类型（comm 类型、进程名排序） | `comm-types.go`、`process-name-sorter.go` |
| `util/` | 通用工具函数 | `util.go` |
| `webgui/` | 嵌入式 Web UI（HTML/CSS/JS，`go generate` 打包） | `index.html`、`js/`、`css/` |
| `xmlrpcclient/` | supervisorctl 使用的 XMLRPC 客户端 | `xmlrpc-client.go`、`xml_processor.go` |
| `pidproxy/` | pidproxy 辅助进程（转发信号） | `pidproxy.go` |

## 平台差异（修改时必须注意）

- **按平台分文件的模式**：`xxx.go` + `xxx_linux.go` + `xxx_windows.go` + `xxx_darwin.go` 等。
  修改跨平台代码时，要同步检查各平台实现（如 `daemonize.go`、`rlimit*.go`、`zombie_reaper*.go`、
  `set_user_id*.go`、`pdeathsig_*.go`、`signal_*.go`、`log_*.go`）。
- Windows 上部分功能退化为空实现（如 `zombie_reaper_windows.go` 只有占位）。
- 新增平台相关文件时，注意 Windows 文件不能依赖 `syscall` 中 Unix 专属常量。

## 关键约定

- **go:generate**：`assets_dev.go` / `assets_release.go` 用 `//go:generate` 嵌入 webgui 资源，
  修改 `webgui/` 下的前端文件后必须重新执行 `go generate` 再编译，否则改动不生效。
- **构建标签**：`-tags release` 决定嵌入的是 release 还是 dev 资源。
- **单模块结构**（renew 分支已合并）：子包（config/process/events 等）已不再是独立 module，
  统一由根 `go.mod` 管理，`go test ./...` 覆盖全部包。import 路径保持 `github.com/ochinchina/supervisord/<pkg>` 不变。
- **配置兼容性**：必须保持与 Python supervisor 配置文件格式的兼容（`[program:x]`、`[supervisord]`、
  `[inet_http_server]` 等 section）。当前正在向 TOML 迁移（TOML 优先 + INI 兼容，见 RENEW_PLAN.md）。
- **事件协议**：`events/` 实现的 EVENT 协议需与 Python supervisor 的事件监听器兼容。

## 测试

- 全量测试：`go test ./...`（单模块结构，覆盖所有包）
- 静态检查：`go vet ./...`（必须零告警）
- 已有测试参考：`content_checker_test.go`、`process/process_manager_test.go`、`config/config_test.go`、
  `logger/log_test.go`、`events/events_test.go` 等。

## 项目 Renew 计划

- `RENEW_PLAN.md` — 项目目标、决策记录（ADR）、分阶段任务与验收标准（单模块合并/依赖升级/Windows 审计/TOML 迁移/AI Skills）。
- 开发在 `renew` 分支进行，逐 Phase 验收后合并回 master。

## 常见任务指引

1. **新增一个管理命令**（如 `supervisorctl` 子命令）→ 看 `ctl.go` 中的 `StatusCommand`/`StartCommand` 模式，
   用 `go-flags` 定义结构体，并在 `supervisor.go` 的 RPC 方法中实现。
2. **修改进程生命周期逻辑**（启动/停止/重启/自动重启）→ 看 `process/process.go` 和 `process/process_manager.go`。
3. **新增配置项** → 在 `config/config.go` 解析，`config_template.go` 有模板，`supervisor.ini` 有示例。
4. **修改 Web UI** → 改 `webgui/` 下文件，然后 `go generate && go build -tags release`。
5. **修改日志轮转** → `logger/log.go`。
6. **修改 XMLRPC 接口** → `xmlrpc.go` + `faults/faults.go`（错误码）。

## 提交规范（.github 中有 CI 配置）

- 提交前运行 `go vet ./...`（各模块）与 `go test`，确保通过。
- `.goreleaser.yml` 定义了发布流程；`Dockerfile` / `Dockerfile.github` 定义镜像构建。
