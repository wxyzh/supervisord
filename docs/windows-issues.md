# Windows 平台问题清单

> 来源：GitHub 上游仓库 `ochinchina/supervisord` issues + 代码审计。
> 用途：Phase 2（Windows 审计与修复）的工作素材，逐条跟踪。
> 状态图例：⬜ 待审计 · 🔄 审计中 · ✅ 已确认根因 · 🔧 已修复

## 1. GitHub Issues 汇总

### 1.1 服务注册/启动类（高优先级）

| Issue | 标题 | 状态 | 分析 |
|---|---|---|---|
| [#360](https://github.com/ochinchina/supervisord/issues/360) | 注册 windows 服务后，无法正常运行 | ⬜ | 与 service 注册（kardianos/service）相关 |
| [#361](https://github.com/ochinchina/supervisord/issues/361) | Windows11 & Windows Server 注册系统服务均出现启动 1053 错误 | ⬜ | 1053 = 服务未及时响应启动；kardianos/service 已升级 v1.3.0，需验证 |
| [#311](https://github.com/ochinchina/supervisord/issues/311) | Windows10 & Windows Server 注册系统服务均出现启动 1053 错误（重复） | ⬜ | 同上 |
| [#264](https://github.com/ochinchina/supervisord/issues/264) | Win10 下注册系统服务失败的问题 | ⬜ | 同上 |

### 1.2 进程停止/信号类

| Issue | 标题 | 状态 | 分析 |
|---|---|---|---|
| [#416](https://github.com/ochinchina/supervisord/issues/416) | stopwaitsecs and taskkill in Windows | ⬜ | Windows 下超时停止是否调用 taskkill /T 杀进程树？`process.go` 的 stop 逻辑 |
| [#135](https://github.com/ochinchina/supervisord/issues/135) | Windows Error: reaper.go:23:22: undefined: syscall.SIGCHLD | ⬜ | 已修复（zombie_reaper_windows.go 空实现），需确认现在行为 |

### 1.3 WebGUI / 构建类

| Issue | 标题 | 状态 | 分析 |
|---|---|---|---|
| [#362](https://github.com/ochinchina/supervisord/issues/362) | winserver2016 webgui 404 导致 ctl 无法使用 | ⬜ | `go generate` 未执行导致 assets 未嵌入；文档需说明 + 校验 |
| [#363](https://github.com/ochinchina/supervisord/issues/363) | windows 打包方式：直接 go build 访问 webgui 404 | ⬜ | 同 #362：必须 `go generate` + `-tags release` |
| [#199](https://github.com/ochinchina/supervisord/issues/199) | Releases 中只有 linux 编译版本 | ⬜ | goreleaser 已支持 windows，属发布流程问题 |
| [#328](https://github.com/ochinchina/supervisord/issues/328) | Could you release up to date windows binary? | ⬜ | 同上 |

### 1.4 其他

| Issue | 标题 | 状态 | 分析 |
|---|---|---|---|
| [#239](https://github.com/ochinchina/supervisord/issues/239) | Fail to compile the supervisor under windows platform | ⬜ | 历史编译问题，需复测 |
| [#375](https://github.com/ochinchina/supervisord/issues/375) | windows 下有其他的守护工具可以使用吗 | ⬜ | 咨询类，非 bug |

## 2. 代码审计清单（`*_windows.go` 与平台差异点）

| 文件 | 职责 | 审计结论 | 状态 |
|---|---|---|---|
| `daemonize_windows.go` | 守护进程化（Windows 下行为） | 待审计 | ⬜ |
| `zombie_reaper_windows.go` | 僵尸进程回收（Windows 空实现） | 待审计 | ⬜ |
| `signals/signal_windows.go` | 信号名→Windows 信号（CTRL_C/CTRL_BREAK）映射 | 待审计 | ⬜ |
| `process/pdeathsig_windows.go` | 子进程随父进程死亡（Windows 不支持） | 待审计 | ⬜ |
| `process/set_user_id_windows.go` | 切换运行用户（Windows 不支持） | 待审计 | ⬜ |
| `logger/log_windows.go` | 日志（Windows 控制台/文件差异） | 待审计 | ⬜ |
| `rlimit_windows.go` | 资源限制（Windows 空实现） | 待审计 | ⬜ |
| `pidproxy/signal_windows.go` | pidproxy 信号转发（Windows） | 待审计 | ⬜ |
| `process/process.go`（通用） | 停止逻辑：Windows 下超时是否需 `taskkill /T`（关联 #416） | 待审计 | ⬜ |

## 3. 修复优先级建议

1. **P0**：#360/#361/#311/#264 服务注册失败（kardianos/service v1.3.0 升级后复测）
2. **P0**：#416 Windows 下进程树停止（`taskkill /T`）
3. **P1**：#362/#363 webgui 404（文档 + 构建校验，可加 CI 检查 go generate 产物一致）
4. **P2**：剩余审计项

> 本清单由 `gh` 拉取（2026-xx 更新），审计结论随 Phase 2 推进持续补充。
