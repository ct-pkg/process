# process

POSIX 子进程执行包：不经 shell 启动一个子进程，分别捕获 stdout / stderr，
返回退出码、信号、超时与启动失败等可区分状态。

- 版本：0.1.0
- 许可：MIT
- 平台：macOS 与 Linux（POSIX）；Windows 本轮不做
- 状态：**需求层梳理完成，未立项**。本仓没有实现、没有 phase 卡。
- 需求：[docs/requirements.md](docs/requirements.md)
- 待办事项：[docs/backlog.md](docs/backlog.md)
- 会话映射：[docs/session-map.md](docs/session-map.md)

## 定位与边界

- 进：启动、参数逐字节传递、环境变量、工作目录、stdin 策略、stdout / stderr
  分别捕获、退出码与信号、墙钟超时、超时后的进程组终止、并发上限、错误分类。
- 不进：shell 语法解析、PTY / 终端仿真、作业控制、进程间通信协议、异步事件循环、
  容器 / cgroup / 资源限制、Windows。
- 安全默认：不调用 `system()` / `popen()`，不经过 `/bin/sh`；参数不做词分割、通配或变量展开。

## 门禁

需求阶段只落了规约生成块，**没有** `tools/check.sh`；因此不存在可以伪装通过的 g0–g9。
当前唯一可执行的校验是：

```sh
bash /Users/samxx/trae-ws/.ai-engineering/bin/sync-agents.sh --check AGENTS.md
```

立项后按 `AGENTS.md` §6.4 建 `tools/check.sh`，未实现的子命令必须显式退出 2。
