# 会话映射

| 类型 | 名称 | 负责 | 产物 |
| --- | --- | --- | --- |
| 常驻 | 需求池 | REQ | `issues/`、phase 立项 |
| 常驻 | 技术池 | ARCH | `docs/design.md`、battle 记录 |
| 常驻 | 评审 | REVIEW | 复核结论、`verified` 证据 |
| phase | `[ct-process] PH-<n> · <目标一句话>` | DEV / VERIFY | `docs/phases/PH-<n>/card.md` 的收口对照表 |

## 阶段 → 会话

| 阶段 | 目标 | 会话标题 | 状态 |
| --- | --- | --- | --- |
| 需求 | CT 能力边界核实 + 需求规约 | `[ct-process] 需求梳理 · 子进程执行` | 本轮完成，未立项 |

## 约定

- 一个 phase 一个会话，不复用旧会话；标题按上表。
- 任务级清单写在 phase 卡里，不在本文件重复。
- 卡片冻结后任务清单不得中途修改（T-6）。
