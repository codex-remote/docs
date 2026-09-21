---
title: Mobile Web 前端视觉与动效规范
date: 2026-08-25
updated: 2026-08-25
tags:
  - ai-coding-remote
  - mobile-web
  - frontend
  - visual-design
  - motion
status: current
implementation_status: implemented-baseline
related:
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
  - "[[Runtime JSON 轮询传输架构]]"
  - "[[Run Server HTTPS 与 SSE 接口规范]]"
---

# Mobile Web 前端视觉与动效规范

> [!note] 目标
> Mobile Web 是高频操作的 Codex 远程工作台。视觉应优先支持阅读、判断和恢复，不把后台事件数量直接等同为屏幕动画数量。

## 1. 视觉方向

- 基调：安静、清晰、克制的工程工作台；内容优先于装饰。
- 层级：页面背景、会话内容、用户消息、助手结果、活动状态、诊断检查器各自有稳定层级。
- 密度：移动端优先保证单手阅读和输入，桌面端使用更宽的阅读列，但不堆叠无关卡片。
- 形状：重复项使用轻边界和小圆角；不使用大面积营销式卡片、浮夸渐变或装饰性光球。
- 文案：展示用户任务和当前状态；不把底层协议、重试实现或模型控制词汇暴露为产品文案。

## 2. 状态与布局稳定性

任何高频 Runtime 状态都必须先经过展示状态层，再进入 UI：

```text
Runtime event
  -> domain state / durable cursor
  -> presentation state (coalesced, delayed, bounded)
  -> stable layout
```

规则：

- 事件到达不能直接创建无限增长的可见区域。
- 加载、刷新、失败、终态必须使用固定或可预测的高度。
- 流式回答的正文增长可以改变内容高度，但活动栏、状态标签和轨迹摘要不能反复插入/移除造成跳动。
- 背景刷新不应把已稳定的 `ready` 页面切换成可见 `refreshing`，除非用户明确触发刷新。
- 任何滚动引导都必须尊重用户滚动意图，不因后台事件重新抢回滚动控制权。

## 3. Streaming Activity 规范

当前活动栏是固定高度的单行状态区域，推荐状态：

| 阶段 | 展示 | 动效 |
| --- | --- | --- |
| `preparing` | 正在准备回复 | 稳定占位，不闪烁 |
| `running` | 正在执行/查询/搜索… | 图标轻微旋转，文案交叉淡入 |
| `completed` | 已完成当前步骤 | 绿色完成图标，短暂保持 |
| `composing` | 正在生成回答/整理结果 | 只有真实停顿时显示，至少保持短窗口 |

状态切换规则：

- 新工具状态先合并约 `120ms`，避免连续 `item.started` 逐个驱动布局。
- 当前文案至少保持约 `420ms`，避免完成/下一步之间出现闪现。
- “正在整理结果”延迟出现，并在出现后保持短窗口；如果下一步很快到达，直接切换，不显示中间态。
- 文案只在活动栏内部替换，不改变活动栏外框尺寸。
- `prefers-reduced-motion: reduce` 下取消位移和循环动画，只保留状态、颜色和图标变化。

实现基线见 `mobile-web/src/components/useActivityPresentation.ts`。

## 4. Execution Trace 规范

- “执行轨迹”是可展开的辅助信息，默认不因每个事件自动展开。
- 摘要只展示稳定计数，例如“已完成 4 项操作”或“2 项完成 · 1 项进行中”。
- 展开后使用固定高度滚动窗口；新增步骤在窗口内部进入，不推动整段对话。
- 步骤行只使用轻微透明度/位移动效，不使用高度动画、弹跳或逐行大幅位移。
- 步骤的完整标题、类别、detail 和状态仍然保留；视觉批处理不得丢弃领域事件。
- 当前步骤使用 amber；完成使用 green；失败使用 red；中断使用 amber/neutral，不让颜色成为唯一语义。

## 5. 传输与 UI 的关系

SSE 和 JSON Poll 都输出同一个 `RuntimeEvent`，因此视觉状态机不区分传输方式：

- Poll 空超时不能触发可见同步提示；它只在 Transport 内部续接。
- Session 真实事件可以触发静默后台快照刷新。
- 活跃 Run 的 delta 优先更新当前内存消息；滞后快照不得覆盖正在增长的回答。
- 终态事件和终态快照可以更新完成状态、耗时和轨迹收敛。

## 6. 可访问性与移动端

- 活动栏使用 `role="status"` 和稳定、短的 `aria-label`；不要每个 delta 都触发屏幕阅读器播报。
- 执行轨迹摘要可通过键盘和 VoiceOver 展开/收起；`details/summary` 状态必须保持可用。
- 颜色、图标和文字共同表达运行/完成/失败，满足非颜色识别。
- 390×844 视口下活动栏不横向溢出，步骤 detail 最多两行截断。
- Reduced Motion 下不执行持续扫描动画和滚动引导动画。

## 7. 验收清单

- [x] 高频工具事件不会逐条推高正文布局。
- [x] “正在整理结果”不会在工具切换间隙闪现。
- [x] 空 Poll 超时不会出现同步提示或滚动跳动。
- [x] 活跃 delta 不会被滞后快照覆盖。
- [x] 执行轨迹展开区在桌面和移动视口保持固定高度。
- [x] SSE 传输仍使用同一视觉状态机。
- [x] Reduced Motion、键盘和 VoiceOver 状态有明确退化路径。
