# Agent Note: Folded Workspace Running Activity

Status: implemented

[English](2026-08-31-folded-workspace-running-activity.md) | 中文

## Problem

折叠的 Workspace 分组不渲染任何 Session 行，因此为节省侧边栏空间而折叠分组的用户会丢失通常用于表示运行中会话的那个旋转指示器。只看一眼折叠后的树，无法得知其中某个 Workspace 内的工作仍在进行。

## Decision

`deriveGroups` 为每个分组新增一个派生事实 `GroupNode.hasRunningActivity`，在折叠投影丢弃行列表之前，从每个可见成员 Session 摘要计算得出：当任一成员运行自身的循环（`running`）或存在正在运行的 subagent 后代（`runningSubagentCount > 0`）时为 true。折叠的 `ProjectRowItem` 会在该事实为 true 时，在目录名正前方渲染一枚 `ongoing` 旋转指示器；展开的分组保留逐行指示器，头部不渲染。

提升后的旋转指示器复用与 Session 行相同的 `StateDot` ongoing 呈现（蓝色像素追逐矩阵），并用同一 `status.running` 本地化文案作为其视觉隐藏标签，因此无需向字典新增文案。它仅以折叠状态为开关：分组展开时头部保持干净，行级圆点仍是活动呈现的唯一来源。派生仍保持纯投影——`hasRunningActivity` 是布尔事实，而非渲染器执行的扫描。

## Alternatives considered

**从行组件扫描已渲染的行。** 折叠的分组没有任何子行，组件无法检查它已不再渲染的内容；该事实必须在成员列表仍然存在的地方派生。

**提升所有可能的状态（待处理交互、完成提醒）。** 本次需求是运行中旋转指示器；审批、提问与计划待审的等待状态是另一回事，仍只在分组展开后才可见（包 README 保留了该限制）。

**在头部始终显示旋转指示器，即使展开时也显示。** 这会在文件夹旁重复行级圆点，并为没有信号增益的布局增加噪音；头部指示器的存在只是为了替代那些未渲染的行。

## Consequences

- 折叠树的用户无需展开即可一眼看出某 Workspace 仍在工作；当最后一个运行中的成员（或后代）停止时，旋转指示器随即消失。
- 该聚合与五行的「展开其余」溢出上限无关：`hasRunningActivity` 反映完整的可见成员集合而非渲染切片，因此由工具栏折叠或由单分组箭头折叠的分组都会报告相同的事实。
- 真实 Workspace 分组与 Ungrouped 桶共享 `buildGroup` 里的同一代码路径，因此二者都获得该指示器，无需特例。
- 无需新增本地化文案或 store 状态：该事实是派生字段，文案复用 `status.running`。

## Testing

派生测试覆盖自身运行与后代运行的分组——包括渲染行已清空的折叠分组——以及空闲分组保持 false。行测试覆盖旋转指示器仅在分组处于折叠且运行中时出现在目录名正前方，并在分组展开或停止运行后消失。

## Related

- [Workspace Sidebar Order and Folding](2026-08-11-workspace-sidebar-order-and-folding.zh.md)