# ct-process 需求规约

> 状态：需求层梳理完成，**未立项**。本文只把子进程执行拆成可判定的需求，不做设计选型、不建 phase 卡、不写实现、不改 `ct-lang`。
> 依据：本仓为空仓（`main` 零提交）；CT 能力取证来自 `ct-lang` HEAD（2026-11-14）的文档、夹具与源码；POSIX 行为取证来自本机 macOS 27.2 的独立 C 探针（`/tmp/ctp_probe.c`，未入库）。
> 口径优先级：本文冻结的需求口径 > 同批包（`ct-http` / `ct-sse` / `ct-fsguard`）的写法惯例 > 原生 POSIX 文档。提案原文若存在，保持不动。

## 0 摘要

| 项 | 数量 |
| --- | --- |
| 需求条目 | 29（R-P-01 … R-D-03） |
| 待确认 | 14（Q-01 … Q-14） |
| 对 CT 语言的硬阻塞 | 0 |
| 软阻塞 | 4（无进程封装、无内建超时、本机无 `ct` 无法复跑 CT 编译探针、`posix_spawn` 设 cwd 的平台回退） |
| 原假设判定"不成立 / 部分不成立" | 6（A-01 … A-06，其中 A-06 为部分不成立） |

**判据形态约定（全文适用）**：

1. 每条至少有一条**运行断言**（退出码 / 返回值 / 字节比较 / 状态字段）或**可核对产物**（grep 计数、符号表、字段清单）。
2. **"能编译通过"不单独构成判据**；涉及 API 形态的需求，编译只是前置条件，必须在运行时断言行为。
3. 夹具一律运行时生成，不依赖仓库内大文件；路径按 POSIX `/` 处理。
4. 负数用例必须断言**具体错误分类与错误内容**，不得只断言返回 `-1` 或非 0。
5. 本文的 `test/run.sh <case>` 是验收命令的占位名；实现落地时必须给出真实命令与原始输出。

**术语**：**子进程** = 本包直接创建并等待的进程；**进程组** = 子进程被放入的独立 `pgid`；**捕获** = 从管道读出 stdout / stderr 并保存为字节串；**截断** = 达到输出上限后继续排空但不再保存；**结束状态** = 正常退出 / 信号 / 超时 / 启动失败 / 本地错误之一。

## 1 一句话需求

在 POSIX 上启动一个**不经 shell** 的子进程，把 stdout / stderr 分别捕获为字节串，返回可区分的结束状态（正常退出码 / 信号 / 超时 / 启动失败 / 本地错误），并保证超时后进程组被终止、管道被关闭、子进程被回收。

## 2 CT 能力边界核实

### 2.1 现有能力与缺口

| 能力 | 结论 | 证据 |
| --- | --- | --- |
| `std::fs` / `std::str` / `std::io` / `std::math` | 存在，均由 `ct-pkg/std` 以 `.m.ct` + `.shim.h` 映射 | `ct-lang/docs/tutorial.md:508-521` 的模块表；`ct-lang/docs/glossary.md:222` |
| 内置层范围 | `lib/std/` 只承载 `Vec` / `Map` / `Option` / `Result` / `Str`，不是完整 `std` | `ct-lang/lib/std/README.md:1-10` |
| `std::process` 或等价进程封装 | **不存在** | 模块表（`tutorial.md:508-521`）无 process / os / sys 项；全工作区检索 `std::process`、`Process.m.ct`、`process.m.ct` 只在 `ct-fsguard` 的两处文字里提到 `ct-process`，没有实现 |
| 内建超时 / 定时器 | **不存在** | `ct-lang/docs/language-spec.md` 检索 `timeout` / `alarm` / `timer` / `sleep` 无命中；`ct-lang/lib/` 同样无命中 |
| C 绑定机制 | `.m.ct` + 同目录 `<base>.shim.h/.c`；shim 头由 `ct build` 自动 include | `ct-lang/test/bindings/README.md:56-84`；`ct-lang/docs/package-guide.md:16-38` |
| POSIX libc 可绑定 | 已用 `socketpair` / `write` / `read` / `errno` 证明；指针出参、`size_t` 转换、`errno` 全局量都放在 `static inline` shim 里 | `ct-lang/test/bindings/posix-socket/posix/socket.m.ct:1-15`、`socket.shim.h:1-27` |
| CT 单元直接 include C 头并调 libc | 已支持；编译器自身就 `#include <unistd.h>` / `<stdlib.h>` 并调 `system()` | `ct-lang/selfhost/ct-ast/frontend/module/Module.h.ct:19-23`、`:1834-1841` |
| 函数指针 / 跨单元 typedef | 已支持；回调必须是具名函数或函数指针，闭包字面量不能当 C 回调 | `ct-lang/docs/bindings-and-patterns.md:185-236`；`ct-lang/test/ast/27_fnptr_forms/prog.ct:9-22` |
| `pub enum` 常量 | 已支持；成员进入依赖单元生成头，裸名与限定名都可用 | `ct-lang/docs/language-spec.md:748` |
| 显式 `Drop` + `delete` | 已支持；显式 `impl Drop` 时不生成字段递归 glue | `ct-lang/lib/std/Str.h.ct:21-27`；`ct-lang/docs/bindings-and-patterns.md:311-368` |
| `atomic T` 与 `ct_atomic_*` | 已支持（Phase 41）；用于并发计数可避免额外 C 依赖 | `ct-lang/docs/language-spec.md:2277-2292` |
| 语言级线程 / 线程池 | **不存在**；线程由调用方直接 `#include <pthread.h>` 创建，CT 不做 `Send` / `Sync` 检查 | `ct-lang/docs/modern-oo-gap.md:205`；`ct-lang/docs/design/phase41-draft.md:58` |

**结论**：CT 没有现成的进程封装，也没有内建超时；但**绑定路径是通的**，不需要新增语言关键字或语法。进程封装与超时都必须由本包在 `.m.ct` + `.shim.h` 里自建。

### 2.2 POSIX 机制实测（macOS 27.2，Apple clang 21.0.0）

探针用 `posix_spawn` + `pipe` + `poll` + `waitpid` + `kill` 复现本包要用的五条路径；完整源码在 `/tmp/ctp_probe.c`，未入库。原始输出：

```text
[1] timed_out=0 status=0x700 exited=7
stdout=OUT stderr=ERR
[2] timed_out=1 status=0x9 signaled=9
elapsed_ms=201
[3] timed_out=0 status=0x0 exited=0
stdout=/private/tmp|hello
[4] timed_out=0 status=0x0 exited=0
stdout_len=2001 last=y
[5] timed_out=0 status=0xf signaled=15
```

对应结论：

- `[1]` stdout 与 stderr 可分别捕获到 `OUT` / `ERR`，正常退出码 7 可解出。
- `[2]` 200 ms 超时后杀进程组，`waitpid` 解出 `SIGKILL(9)`，实际耗时 201 ms；墙钟超时可行。
- `[3]` 自定义环境变量 `CT_PROBE=hello` 生效；`cwd=/tmp` 在子进程里显示为 `/private/tmp`（OS 解析符号链接后的路径）。
- `[4]` 2001 字节输出在 `poll` 循环里完整读出，输出期间穿插 `sleep` 不死锁。
- `[5]` `SIGTERM(15)` 与正常退出可区分。

**边界**：探针是 C，不是 CT 构建；本机 `which ct` 无输出，所以 CT 侧的编译/运行未复跑。CT 能力结论以 §2.1 的文档与夹具为准，落地时必须用真实 `ct` 再跑一遍。

### 2.3 平台与实现边界

- `posix_spawn_file_actions_addchdir` 在 macOS SDK 存在（`/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/spawn.h:72`）；`addchdir_np` 仍在但已标记 deprecated（`:184-185`）。Linux glibc 2.29+ 提供 `_np` 版本，更老的 glibc 需要 `fork` + `exec` 回退。**本机只验证了 macOS**。
- `posix_spawn` 在多线程进程里比 `fork` + `exec` 安全；但 `fork` + `exec` 是 cwd / 环境变量 / 进程组配置的通用回退。实现选型不在本文冻结，只要求语义一致（Q-09）。
- 本包**不经过 shell**：`system()` / `popen()` 明确禁止。编译器自身用 `system()`（`Module.h.ct:1834`）只作为"CT 能调 libc"的证据，不作为本包的实现方式。

## 3 使用边界

| 角色 | 说明 |
| --- | --- |
| 上游 | CT 调用方：传入可执行文件、参数、环境、工作目录、超时与输出上限 |
| 下游 | POSIX 内核：`fork` / `posix_spawn`、`pipe`、`poll`、`waitpid`、`kill` |
| 不属于本包 | shell 语法、PTY、作业控制、进程间通信协议、异步事件循环、容器/资源限制、Windows |

## 4 需求条目

### 4.1 P 组：包形态与平台

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-P-01 | 仓库根有 `ct.pkg`、`LICENSE`、`README.md` | `name=process`、`version=0.1.0`、`licence=MIT`、`requires=ct >= 0.34.0`；不依赖外部 C 库；`ct get process@^0.1.0` 与 `ct install` 可识别 | ① `ct.pkg` 五个键逐个 grep 到；② 干净目录 `ct get` + `ct install` 退出 0；③ `otool -L`（macOS）/ `ldd`（Linux）只指向系统 libc；④ 删 `name` 后 `ct get` 非零退出并报缺键 | 非目标：不 vendored libc、不提交索引行、不打 tag。依赖：`ct-lang/docs/package-guide.md:64-84` |
| R-P-02 | 在 macOS 与 Linux 构建运行；在 Windows 构建 | macOS 与 Linux 支持；Windows 不构建、不宣称支持；源码无 Win32 分支 | ① 两平台 `test/run.sh all` 原始输出均 0 失败；② `grep -ri win32` 只出现在文档；③ 仓库内无 `win32/` 实现目录 | 非目标：Windows / Cygwin / WSL 特殊处理。依赖：本机只有 macOS，Linux 需 CI 或第二台机器 |
| R-P-03 | 任何 POSIX 进程调用 | `fork` / `exec*` / `posix_spawn*` / `waitpid` / `pipe` / `poll` / `kill` 只出现在 `.m.ct` recipe 与 `*.shim.h/.c`；公开头不暴露 POSIX 类型，不 include POSIX 头 | ① 排除 `.m.ct`、`*.shim.h/.c` 后，对 `fork`、`execv`、`posix_spawn`、`waitpid`、`poll(`、`kill(`、`pipe(` 逐个 grep，`src/*.ct`、`*.h.ct` 中命中数为 0；② 生成公开头无 `<sys/wait.h>` / `<unistd.h>`；③ `nm -g` 生成对象不导出包内 shim helper（helper 全 `static` / `static inline`；系统 libc 符号可以存在） | 非目标：不接其他 C 库、不做动态加载。依赖：`test/bindings/README.md:56-84` |

### 4.2 S 组：启动与会话

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-S-01 | 可执行文件 + argv 数组，含空格、空串、`$HOME`、`*`、`;`、UTF-8、超长参数 | 不经 shell；argv 逐字节传递；空串保留；不做词分割、通配、引号去除、变量展开；`argv[0]` 按调用方给定，未给时以实现写入的可执行文件路径填充并在文档写明 | ① 夹具打印 `argc` 与每个参数的十六进制字节，逐项比对；② 参数 `; touch /tmp/pwned` 作为单个参数时不创建文件；③ 不存在的可执行文件返回启动错误且子进程数为 0 | 非目标：shell 语法、命令替换、别名、内建命令。依赖：`posix_spawn` 或 `fork` + `exec` |
| R-S-02 | `"/bin/echo"`、`"./x"`、`"echo"`、空串、`PATH` 含空项或相对项 | 含 `/` 按路径解析；不含 `/` 按 Q-08 冻结的 `PATH` 来源搜索（建议：子进程环境的 `PATH`）；`PATH` 为空则找不到；目录或不可执行文件返回启动错误；无 shell 回退 | ① `"echo"` 可运行；② `"definitely-not-a-command"` 返回 `PROC_ERR_SPAWN` + `ENOENT`；③ `"/bin"` 返回 `EACCES` 或 `EISDIR`；④ `PATH=""` 下 `"echo"` 返回 `ENOENT`；⑤ 负例：实现走 shell 时 `echo` 内建会成功，断言必须变红 | 非目标：命令哈希、shell 内建、别名。依赖：`posix_spawnp` 或手写 `PATH` 搜索 |
| R-S-03 | 默认继承；`env_clear()`；`env_set("K","V")`；重复键；无 `=` 的条目 | 默认继承父进程环境（spawn 时刻快照）；`env_clear` 后为空环境；`env_set` 逐字节写入，值可为空、可含 `=`，不得含 NUL；重复键后者覆盖；非法条目在 spawn 前返回本地错误 | ① 子进程打印环境，断言继承标记存在；② clear 后父标记不存在；③ 只 set 时恰好只有 `K=V`；④ 重复键取最后值；⑤ 非法条目返回 `PROC_ERR_INVALID` 且子进程数为 0 | 非目标：`$VAR` 展开、dotenv、修改父进程环境。依赖：`envp` |
| R-S-04 | 默认 cwd；绝对 cwd；相对 cwd；不存在的 cwd；符号链接 cwd | 默认继承父 cwd；显式 cwd 在 exec 前生效；相对 cwd 相对父 cwd 解析；不存在的 cwd 返回启动错误且子进程数为 0；父进程 cwd 不变 | ① 子进程打印 `getcwd`，与期望一致；② `cwd=/tmp` 时子进程打印 `/private/tmp`（macOS 实测）；③ 不存在的 cwd 返回 `PROC_ERR_SPAWN`；④ 调用前后父进程 `getcwd` 相同 | 非目标：chroot、namespace、umask。依赖：`posix_spawn_file_actions_addchdir` / `fork` + `chdir`；Linux 旧 glibc 需回退 |
| R-S-05 | 默认 stdin；可选的 stdin 字节（Q-05） | MVP 子进程 stdin 指向 `/dev/null`，`read` 立即返回 0，不继承父进程终端；MVP 不提供 stdin 写入。若 Q-05 纳入写入：逐字节写入、写完关闭、`EPIPE` 报写错误但子进程结果仍返回 | ① 子进程 `read(0,buf,1)` 在 1 秒内返回 0；② 父进程终端输入不被消费；③ 若纳入写入：子进程收到含 NUL 的完整字节；EPIPE 用例返回 `PROC_ERR_IO` 且 `kind` 仍可读 | 非目标：PTY、交互式 stdin、终端控制。依赖：Q-05 |

### 4.3 C 组：输出捕获

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-C-01 | 子进程写 stdout：0 字节、`a\0b`、1 MiB | 结果对象含 stdout 缓冲与字节长度；逐字节一致；不裁剪、不转码；空输出长度为 0；内嵌 NUL 保留 | ① `printf 'a\0b'` 后 `len==3` 且 `memcmp` 相等；② 1 MiB 与参考文件逐字节相等；③ 空输出 `len==0`；④ 负例：用 `strlen` 计数时 NUL 用例变红 | 非目标：按行切分、UTF-8 校验、trim。依赖：`pipe` + `read` |
| R-C-02 | 子进程分别写 stdout 与 stderr | 两个缓冲完全独立；不合并；不串流；不承诺两路之间的相对顺序 | ① 子进程 stdout 写 `O`、stderr 写 `E`，两个缓冲各自逐字节相等；② 两路各写 1 MiB 时都完整且无交叉；③ 负例：合并成一路时独立断言变红 | 非目标：合并输出、保留交错顺序。依赖：两条独立 `pipe` |
| R-C-03 | 子进程在退出前向 stdout 与 stderr 各写超过管道容量（如各 1 MiB） | 两条管道被并发排空；子进程能写完并退出；调用不超时、不死锁；两个缓冲都完整 | ① 两路各 1 MiB、中间 `sleep 0.1`，调用在 5 秒内返回且长度精确；② 负例：先读 stdout 再读 stderr 的顺序实现必须死锁（记录红证） | 非目标：不保证两路顺序、不做合并。依赖：`poll` / `select` 或内部线程 |
| R-C-04 | 一次性结果；可选的流式回调（Q-06） | MVP 在子进程退出后一次性返回完整缓冲；实现必须**边运行边排空**，不得先 `waitpid` 再 `read`。若 Q-06 纳入流式：回调在调用内同步触发，按流有序，返回非 0 可中止，返回后不再回调 | ① 一次性：1 MiB 输出在返回后可读；② 流式（若纳入）：1 MiB 至少 2 片，拼接逐字节相等；③ 中止用例返回 `PROC_ERR_CALLBACK` 且子进程被终止并回收；④ 负例：只给 1 片时"至少 2 片"断言变红 | 非目标：异步事件循环、调用方可见线程。依赖：函数指针（§2.1） |
| R-C-05 | 配置输出上限；子进程写出超过上限 | 每条流有独立上限；最多保存 cap 字节；超过后继续排空并丢弃，子进程不被管道阻塞；置 `truncated` 标志；结束状态仍完整返回；内存随 cap 有界；二进制安全 | ① cap=1024、子进程写 1 MiB，`len==1024`、`truncated==1`、退出码 0；② cap=0 时 `len==0`，有输出则 `truncated==1`；③ 分配计数或 RSS 不随输出总量线性增长；④ 负例：无上限实现 RSS 持续增长 | 非目标：落盘 spool、压缩、按行上限。依赖：默认值见 Q-04 |

### 4.4 E 组：退出码与信号

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-E-01 | 子进程正常退出 0 / 1 / 7 / 255 | `kind=exited`；`exit_code` 等于该值；`is_ok()` 仅在退出码 0 时为真；信号字段无效 | ① `sh -c 'exit N'` 表驱动断言；② 退出 1 时 `is_ok()==0`；③ 负例：把退出 1 当成功时断言变红 | 非目标：正常退出不映射成 128+N。依赖：`WIFEXITED` / `WEXITSTATUS` |
| R-E-02 | 子进程被 `SIGTERM` / `SIGKILL` / `SIGABRT` 终止 | `kind=signaled`；`term_signal` 等于信号值；`is_ok()==0`；`exit_code` 无意义并按 Q-10 固定 | ① `kill -TERM $$` 解出 15；② `kill -KILL $$` 解出 9；③ `abort()` 解出 6；④ 负例：把信号报告为退出 143 时断言变红 | 非目标：信号转发、自动重启、core 文件处理。依赖：`WIFSIGNALED` / `WTERMSIG` |
| R-E-03 | 正常退出 / 信号 / 超时 / 启动失败 / 本地错误 / I/O 错误 | 结束状态**恰好**属于一个顶层分类；超时独立于"被信号终止"；启动失败与本地错误表示没有子进程运行；非 0 退出不是包错误 | ① 六类各一条夹具，断言恰好一个 `kind`；② 超时用例 `kind=timeout` 且 `timed_out==1`，不得只报 `signaled`；③ 未知 wait 状态返回 `PROC_ERR_INTERNAL`，不得静默吞掉 | 非目标：不设 "unknown" 静默分支。依赖：Q-10 |

### 4.5 T 组：超时与终止

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-T-01 | `timeout_ms`；子进程 `sleep 5`；输出持续但总时长超限；`timeout=0`；负数 | 墙钟总预算，从 spawn 前开始计时，使用单调时钟；不是"空闲超时"，输出活动不重置；0 表示不限（Q-01）；负数在 spawn 前返回本地错误；超时在 `[timeout, timeout+500ms]` 内触发 | ① `sleep 5` + 200 ms 超时，实际返回 201 ms（§2.2 `[2]`）；② 子进程每 50 ms 输出一次仍约 200 ms 超时；③ 负例：空闲重置实现会跑满 5 秒；④ `timeout=0` 等待正常退出 | 非目标：CPU 时间超时、单流空闲超时。依赖：`poll` 超时 + 单调时钟 |
| R-T-02 | 子进程创建同组孙进程；超时 | 子进程放入独立进程组；超时先向进程组发 `SIGTERM`，宽限期后 `SIGKILL`（Q-02）；直接子进程被回收；组内进程不再存活；自行 `setsid` 逃逸的进程不在保证范围，文档写明 | ① `sh -c 'sleep 30 & sleep 30'` + 200 ms 超时后 `pgrep -g <pgid>` 为空；② 直接子进程 `signaled==9`；③ 无僵尸进程；④ 负例：只杀直接子进程时孙进程仍存活，断言变红 | 非目标：cgroup / namespace、追杀已逃逸进程组。依赖：`setpgid` / `POSIX_SPAWN_SETPGROUP` + `kill(-pgid, …)` |
| R-T-03 | 子进程先打印 `before` 再 `sleep`；超时 | `kind=timeout`、`timed_out==1`；已捕获的 stdout / stderr 保留到上限；管道关闭；子进程被回收；无泄漏 | ① stdout 至少包含 `before`；② `timed_out==1`；③ `waitpid` 后无僵尸；④ ASAN / LSAN 退出 0、无报告 | 非目标：超时后重试。依赖：R-T-02 |
| R-T-04 | `poll` 被信号打断；超大 `timeout`；系统时钟调整 | 使用单调时钟；`EINTR` 重试且不延长原 deadline；超大值不溢出，按 Q-01 文档化 | ① 等待期间发送无关信号，仍在约 200 ms 返回；② 静态核对 shim 使用 `CLOCK_MONOTONIC`；③ 超大 `timeout` 用例不出现负值或立即超时 | 非目标：CPU 时间、日历时间。依赖：shim |

### 4.6 O 组：并发上限

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-O-01 | 8 个线程同时调用；上限 2；上限 0；启动失败 / 超时路径 | 最多 M 个子进程同时存活（作用域按 Q-07 冻结；建议：包级）；超出的调用阻塞等待（或按 Q-07 失败）；所有结束路径都释放名额；不因为启动失败或超时泄漏名额；可配置 0 为不限 | ① 8 个 `sleep 0.3`、上限 2，采样 `ps` 子进程数 ≤ 2，总耗时约 1.2 秒；② 结束后计数为 0；③ 启动失败后下一次调用仍能获得名额；④ 负例：忽略上限时观察到 >2 | 非目标：异步队列、优先级、公平性、CPU/内存限制。依赖：CT `atomic`（Phase 41）或 shim 互斥量；CT 无语言级线程，夹具直接 `#include <pthread.h>`，包不自带线程 API |

### 4.7 X 组：错误分类

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-X-01 | 非法参数、`ENOENT`、`EACCES`、`E2BIG`、管道/读写失败、超时、信号 | 稳定分类：`PROC_OK`、`PROC_ERR_INVALID`、`PROC_ERR_SPAWN`、`PROC_ERR_IO`、`PROC_ERR_CALLBACK`、`PROC_ERR_INTERNAL`；spawn / I/O 保留 `errno`；超时是状态而非包错误；非 0 退出不是包错误 | ① 表驱动断言 code 与 errno；② 不存在可执行文件 → `PROC_ERR_SPAWN` + `ENOENT`；③ 退出 1 → `kind=exited` 且无 `PROC_ERR_*`；④ 负例：退出 1 被写成 `PROC_ERR_IO` 时变红 | 非目标：全量 errno 导出、本地化、重试分类。依赖：shim 捕获 `errno` |
| R-X-02 | 同一错误连续触发两次 | 错误文本非空，包含操作与路径 / errno；同一输入可复现；不得指向已释放内存 | ① 两次运行按前缀比对一致；② 文本包含路径与 errno 名；③ `delete` 后不再读旧文本（ASAN 无 UAF） | 非目标：精确文本 ABI、i18n。依赖：Q-14 |
| R-X-03 | `executable` 为 NULL / 空串；`argv` 为 NULL；`argv[0]` 为 NULL；字符串含 NUL；负 `timeout` / 负 cap；非法 env 条目 | 全部在 spawn 前返回 `PROC_ERR_INVALID`；子进程数为 0；不留下半初始化状态；错误文本可读 | ① 非法输入表逐行断言；② 用包装器统计子进程创建次数为 0；③ 负例：先 spawn 再报参数错时计数 >0，断言变红 | 非目标：不替调用方做 shell 引号或清洗。依赖：无 |

### 4.8 L 组：生命周期与所有权

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-L-01 | `Process.new()` → `run()` → `delete`；提前错误；二次 `delete` | `new()` 返回可用句柄；`delete` 关闭管道、释放缓冲、回收子进程；所有错误路径同样清理；二次 `delete` 按冻结口径安全或明确硬错 | ① 正常 / 启动失败 / 超时 / 回调中止四条路径 ASAN + LSAN 全绿；② 二次 `delete` 用例不崩溃（按 Q-11）；③ 负例：漏 `delete` 在 LSAN 下报泄漏，证明检测器开着 | 非目标：引用计数、跨线程共享同一句柄。依赖：显式 `Drop` + `delete` |
| R-L-02 | `run()` 返回结果；再次 `run()`；`delete` 后读结果 | 结果借用句柄，有效期到下一次 `run()` 或 `delete`；不单独释放；文档写明；是否提供深拷贝见 Q-11 | ① `run()` 后立即读结果正确；② 第二次 `run()` 后旧指针内容可变（证明借用）；③ `delete` 后读旧指针在 ASAN 下报 heap-use-after-free | 非目标：长生命周期结果、GC。依赖：Q-11 |

### 4.9 D 组：交付与门禁

| 编号 | 输入 / 触发 | 期望行为 | 验收判据 | 非目标 / 依赖 |
| --- | --- | --- | --- | --- |
| R-D-01 | 仓库与测试 | 至少一个**可运行示例**纳入门禁：运行 `/bin/echo` 与 `sh -c 'printf out; printf err >&2; exit 7'`，断言 stdout / stderr / 退出码；示例失败则门禁失败 | ① 门禁命令退出 0；② 示例输出逐字节匹配；③ 负例：改坏示例后门禁变红，不得返回 0 | 非目标：性能基准、模糊测试。依赖：`tools/check.sh` 立项后建 |
| R-D-02 | README 与需求 / 设计文档 | 文档列出公开 API、所有权契约、超时语义、输出上限、环境与 cwd、平台限制、无 shell 保证、状态分类、已知限制 | ① 文档清单逐项可核对；② 可执行代码块来自 `docs/snippets/` 并由 g4 编译运行；③ 负例：写入不存在的 API 时 g4 变红 | 非目标：教程站、用户手册。依赖：立项后建 snippets |
| R-D-03 | `ct.pkg#version` 与 README 版本行 | 两处相等；干净目录 `ct get process@^0.1.0` + `ct install` 后示例可运行；tag 与 push 只在用户显式指令后执行 | ① g0 版本检查通过；② 干净安装后示例退出 0；③ 负例：`requires` 高于本机 `ct` 时 `ct get` 失败并打印原因 | 非目标：自动推分支 / tag、提交索引行。依赖：用户授权 |

## 5 原假设与判定

| 编号 | 原假设 | 判定 | 证据 | 改后口径 |
| --- | --- | --- | --- | --- |
| A-01 | 用 `system()` / `popen()` 执行命令就够了 | **不成立**：`system` 走 `/bin/sh`，无法逐字节控制 argv，stderr 不能独立捕获，退出码被 shell 包装，且有注入面 | `Module.h.ct:1834-1841` 显示 `system()` 后还要手工解 wait status；`ct.pkg` 禁止 shell | 不用 `system` / `popen`；走 `posix_spawn` 或 `fork` + `exec` + `pipe` |
| A-02 | `system()` 的返回值就是子进程退出码 | **不成立**：返回值是 wait status，需 `WIFEXITED` / `WEXITSTATUS` 解码 | `Module.h.ct:1839-1841`（`st & 0x7f`、`(st >> 8) & 0xff`） | 统一用 `waitpid` + `WIF*` 宏，区分正常退出与信号 |
| A-03 | `popen()` 能同时分别拿到 stdout 与 stderr | **不成立**：`popen` 只给一个流；要分开必须两条 pipe | POSIX `popen` 语义；§2.2 `[1]` 用两条 pipe 分别拿到 `OUT` / `ERR` | 两条独立 `pipe` + `dup2`，`poll` 并发排空 |
| A-04 | `std::fs` 或 `std` 里已有进程封装 | **不成立**：模块表只有 `str` / `fs` / `utf8` / `vec` / `map` / `arena` / `test` | `tutorial.md:508-521`；`lib/std/README.md:1-10` | 本包自建 `Process` 封装 |
| A-05 | 超时可以用 `alarm` + `SIGALRM` 作为唯一机制 | **不成立（作为唯一机制）**：`alarm` 是进程级全局信号，会与调用方/多线程冲突，难以按调用区分 | `language-spec.md` 与 `lib/` 无 timer；§2.2 `[2]` 用 `poll` 超时 + `kill` 实现 201 ms 超时 | 以 `poll` deadline 为主；信号只作为补充，不暴露给调用方 |
| A-06 | `posix_spawn` 在所有 POSIX 上都能设置 cwd | **部分不成立**：macOS 有 `addchdir`；Linux glibc 2.29+ 才有 `_np`，更老版本需要 `fork` + `exec` 回退 | macOS SDK `spawn.h:72`、`:184-185`；本机实测 `[3]` | 实现可选 `posix_spawn` 或 `fork` + `exec`；语义由 R-S-04 冻结，机制不冻结 |

## 6 待确认

> 每条写清：可选口径、建议口径、**不确认会卡住什么**。未确认前按建议口径写判据；确认或否决后只改对应条目。

| 编号 | 待确认项 | 可选口径与建议 | 不确认会卡住什么 |
| --- | --- | --- | --- |
| Q-01 | 超时语义：墙钟还是 CPU；总 deadline 还是空闲重置；`0` 与负数 | 建议：墙钟总 deadline、单调时钟、`0` = 不限、负数 = 本地错误 | 卡 R-T-01 / R-T-04 的全部期望值与 README 语义 |
| Q-02 | 超时后的终止策略：直接 `SIGKILL`，还是 `SIGTERM` + 宽限 + `SIGKILL`；宽限多少 | 建议：`SIGTERM` 后 200 ms 宽限，再 `SIGKILL` | 卡 R-T-02 / R-T-03 的时序断言与"宽限期内的输出是否保留" |
| Q-03 | 是否杀整个进程组；子进程自行 `setsid` 逃逸时是否接受 | 建议：放入独立进程组并 `kill(-pgid)`；逃逸进程明确不保证 | 卡 R-T-02 的验收范围与 README 限制段 |
| Q-04 | 输出上限：默认值；每流独立还是共享；超限截断还是报错/杀进程 | 建议：每流默认 1 MiB，超限截断并继续排空，置 `truncated` | 卡 R-C-05 的断言与内存上限 |
| Q-05 | stdin：MVP 是否需要写入；默认 `/dev/null` 还是继承；是否支持 EPIPE | 建议：MVP 只给 `/dev/null`，不提供写入；P2 再加 | 卡 R-S-05 是否成立、是否要加写接口 |
| Q-06 | 接口同步还是异步；是否提供流式回调；回调能否中止 | 建议：MVP 同步一次性结果；流式回调留 P2 | 卡 R-C-04 的 API 面与验收 |
| Q-07 | 并发上限：包级全局还是每实例；超限阻塞还是立即失败；默认值 | 建议：包级全局、默认 8、超限阻塞、`0` = 不限 | 卡 R-O-01 的语义与线程测试 |
| Q-08 | 环境变量：默认继承；是否支持 `env_clear` + `env_set`；重复键；PATH 用父还是子环境 | 建议：默认继承、支持完整替换、重复后者覆盖、PATH 用子环境 | 卡 R-S-02 / R-S-03 |
| Q-09 | 可执行文件与 cwd 解析：相对路径相对谁；`PATH` 空项/相对项；是否按 realpath 断言 | 建议：相对路径相对子 cwd；`PATH` 空项跳过；断言用 OS 解析后的路径 | 卡 R-S-02 / R-S-04 的期望值 |
| Q-10 | 退出码表示：信号时 `exit_code` 填什么；`is_ok()` 定义；是否暴露 core dump | 建议：`exit_code` 仅在正常退出有效，信号填 -1；`is_ok() = exited && code==0` | 卡 R-E-01 / R-E-02 / R-E-03 |
| Q-11 | 结果所有权：借用句柄还是独立拥有；是否提供深拷贝；二次 `delete` 的语义 | 建议：借用句柄，`delete` 后失效；二次 `delete` 安全置空；不提供拷贝 | 卡 R-L-01 / R-L-02 |
| Q-12 | Windows 是否本轮做 | 建议：本轮不做，README 与 `ct.pkg` 明确 POSIX only | 卡 R-P-02 的平台范围与发布声明 |
| Q-13 | 编译器下限：`0.34.0`（静态固有方法）还是 `0.41.0`（`atomic`）；并发名额放 CT 还是 shim | 建议：先用 `0.34.0`；若并发上限用 CT 原子则抬到 `0.41.0` | 卡 `ct.pkg` 的 `requires` 与 R-O-01 的实现边界 |
| Q-14 | 错误文本是否稳定：精确比对还是前缀比对 | 建议：只保证前缀 + errno 数值稳定，测试按前缀 | 卡 R-X-02 的测试写法与 README 契约 |

## 7 不做什么

| 非目标 | 说明 |
| --- | --- |
| shell 语法解析 | 不做词分割、引号、通配、变量/命令替换、管道、重定向、别名、内建命令 |
| `system()` / `popen()` | 不经过 `/bin/sh`；两者均不进入实现 |
| PTY / 终端仿真 | 不做伪终端、窗口大小、终端控制、交互式 stdin |
| 作业控制 | 不做 `fg` / `bg` / `jobs` / 挂起 / 继续 / 会话领导权管理 |
| 进程间通信协议 | 除 stdin/stdout/stderr 管道外，不做 socket、共享内存、消息队列、RPC |
| 异步事件循环 | 不做后台任务管理器、回调线程、多路复用公开 API |
| 进程树完整管理 | 不做 cgroup / namespace / chroot / rlimit / 沙箱；只保证同组终止 |
| 信号转发与守护进程化 | 不做信号代理、双 fork、daemon 化、自动重启 |
| 输出语义处理 | 不做编码转换、行缓冲、颜色码解析、日志格式化 |
| 重试与调度 | 不做自动重试、负载均衡、任务队列、优先级 |
| Windows | 本轮不做；不建 `win32/` 实现目录，不宣称支持 |
| 修改 `std::fs` 或 CT 语言 | 不改 `ct-pkg/std`，不新增关键字、语法或编译器分支 |

## 8 CT 能力依赖汇总

| 能力 | 本包用途 | 现状 | 证据 | 阻塞等级 |
| --- | --- | --- | --- | --- |
| `.m.ct` + `.shim.h` + `static inline` | POSIX 绑定边界 | 已支持 | `test/bindings/README.md:56-84`；`posix-socket/*` | 已支持 |
| C 头直通（`<unistd.h>` / `<sys/wait.h>` / `<poll.h>`） | 进程、管道、等待、超时 | 已支持 | `package-guide.md:31`；`Module.h.ct:19-23` | 已支持 |
| 函数指针 / `pub typedef` | 流式回调（若 Q-06 纳入） | 已支持 | `bindings-and-patterns.md:185-236`；`27_fnptr_forms/prog.ct:9-22` | 已支持 |
| `pub enum` | 错误分类与结束状态 | 已支持 | `language-spec.md:748` | 已支持 |
| 显式 `impl Drop` + `delete` | 管道与缓冲释放 | 已支持 | `Str.h.ct:21-27`；`bindings-and-patterns.md:311-368` | 已支持 |
| `atomic T` / `ct_atomic_*` | 并发名额计数（若放 CT 侧） | 已支持（Phase 41） | `language-spec.md:2277-2292` | 已支持 |
| 内建超时 / 定时器 | — | **不存在** | `language-spec.md` / `lib/` 检索无命中 | **软阻塞**：用 `poll` + `waitpid` + `kill` shim 绕过 |
| 进程封装 | — | **不存在** | `tutorial.md:508-521`；全工作区检索无实现 | **软阻塞**：本包自建 `Process` |
| 本机 `ct` 编译器 | 复跑 CT 编译探针 | 本机无 `ct` | `which ct` 无输出 | **软阻塞（验证）**：需安装后或 CI 复跑 |
| `posix_spawn` 设 cwd | 工作目录 | macOS 有；Linux 旧 glibc 需回退 | macOS SDK `spawn.h:72`、`:184-185`；本机实测 `[3]` | 软阻塞：可用 `fork` + `exec` 回退 |
| Windows 平台 | — | 本轮不做 | 范围声明 | 非目标 |

**结论：硬阻塞 0 项。** 软阻塞 4 项（进程封装、内建超时、本机无 `ct` 的验证缺口、`posix_spawn` 设 cwd 的平台回退）；前 3 项有自建 shim / CI 复跑的绕行方案，第 4 项有 `fork` + `exec` 回退。均不改语言，不构成对 CT 语言的新需求。

## 9 未纳入本文件的内容

- 目录划分、`Process` 字段、`posix_spawn` 与 `fork` + `exec` 的选型、`poll` 循环实现、缓冲增长策略：属设计层。
- phase 切分、任务清单、退出判据、`version-minor`、roadmap：属立项层。
- 版本号、tag、索引、发布计划：属收口与发布层；本文件不触发 tag 或 push。
- 具体默认值（输出上限、并发上限、宽限期）在 Q-01…Q-14 定稿后写入设计层，不写进需求表的"期望行为"之外。
