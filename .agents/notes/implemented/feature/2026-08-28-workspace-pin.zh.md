# Agent Note: Workspace Pin

Status: implemented

[English](2026-08-28-workspace-pin.md) | 中文

## Problem

工作区列表会增长到超出用户日常触及的少数几个项目。浏览器的两种既有顺序都无法表达"把这一个固定在最上面":持久化注册表顺序只随显式拖拽移动,而按最近更新排序属于 Session 而非 Workspace。用户需要一种持久、单手势的方式,把一个 Workspace 固定在分组树顶部,并且该状态在重启、重连以及对其他条目的手动重排序之后依然成立。

## Decision

置顶状态以可选的 ISO-8601 `pinnedAt` 时间戳存放在工作区记录上;`undefined` 表示未置顶。`Workspace.setPinned(pinned)` 经由实体共享的 mutate 路径写入它,盖上 `updatedAt` 时间戳,并在请求与当前状态一致时不写介质直接返回。置顶从不触碰 `workspaceIds`——持久化显示顺序保持拖拽产生的原样。

链路上新增一个一元动词 `workspace/setPinned({ workspaceId, pinned })`,返回更新后的 `WorkspaceView` 投影,该投影现在携带可选的 `pinnedAt` 字段。`follow()` 流把置顶当作普通记录变更发布:一条 `upsert` 增量,重连的客户端经 baseline 收敛。没有新增流帧类型。

浏览器在渲染时应用置顶:`deriveGroups` 把置顶 Workspace 划分到未置顶之前,每个分区内保持稳定的 Host 顺序,树节点标记每个分组 `pinned`。Workspace 行菜单承载置顶/取消置顶操作(标签随状态翻转),不经对话框直接提交,并在行上保留持久的置顶角标;失败保持非致命的控制台诊断,与重排序拒绝一致。记录 schema 扩展是增量的,`workspace` 领域停留在版本 2,字段存在之前写入的记录解析为未置顶。浏览器本地视图状态的归属划分(持久化顺序 vs 呈现偏好)见 [sidebar order and folding Agent Note](../../archived/feature/2026-08-11-workspace-sidebar-order-and-folding.md);置顶状态归属持久化一侧,而置顶优先的分区仍是渲染规则。

## Alternatives considered

**置顶 = 前插到持久化 `workspaceIds` 顺序。** 已否决:注册表顺序是拖拽拥有的布局。置顶时前插会让取消置顶变得含糊(该行回到哪里?),并悄悄改写用户亲手排好的布局。

**浏览器本地的置顶集合。** 已否决:置顶状态会活在 Host 权威行之外,在不同浏览器与设备间分叉,侧边栏投影也需要第二个真源来与 Host 顺序合并。

**复用 `updatedAt` 的近期性表达显著度。** 已否决:`updatedAt` 随每次被接受的变更移动(重命名、attach、修剪),"最近变更过"不等于"被有意提前"。

## Consequences

旧介质打开不变且无解析失败;领域版本保持 2,不存在迁移——投影缓存会丢弃过期的记录文档,但 `pinnedAt` 是可选字段,没有文档会因此过期。拖拽语义在每个分区内继续成立,而把未置顶 Workspace 拖到置顶 Workspace 之上在视觉上不会驻留——决定渲染位置的是分区,不是 `insertBefore`。分区逻辑位于浏览器树派生中,未来的任何分组界面都必须重复置顶优先的规则,而不是从 Host 顺序继承它。
