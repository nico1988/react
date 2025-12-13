# React Rerender Architecture & Flow

This document summarizes React's rerender architecture, flow, key functions, and implementation details based on the repository source (packages/react-reconciler and packages/react-dom). It's intended for readers who want to deeply understand how React's reconciler schedules, renders, and commits updates.

---

## 1. Overview (High Level)

React's rerender flow can be split into the following stages:

- Schedule (Update Dispatch): Updates start from setState / dispatch / props changes and pass through `scheduleUpdateOnFiber`, `markRootUpdated`, and `ensureRootIsScheduled` to mark roots and lanes as pending.
- Scheduler: `ReactFiberRootScheduler` adds roots that need work to the queue and uses the `scheduler` package (`scheduleCallback`) to schedule render tasks.
- Render: `performWorkOnRoot` -> `renderRootConcurrent` / `renderRootSync` - build the work-in-progress tree (WIP), reconcile children in `beginWork`, and compute the effect list (Placement/Update/Deletion flags) in `completeWork`.
- Commit: `commitRoot` - run before-mutation effects (getSnapshotBeforeUpdate), then mutation (DOM updates), followed by layout effects and passive effects (useEffect). Finally, set `root.current` to the newly committed tree.


  S --> M[markRootUpdated(root, lane)]


### 3.5 Commit Phase
- `commitRoot(root, finishedWork, lanes, ...)` in `ReactFiberWorkLoop.js` and related commit helpers handle the commit phase.
  - Flush pending passive effects (`flushPendingEffects`).
  - Compute `remainingLanes` and call `markRootFinished`.
  - Store `finishedWork` in `pendingFinishedWork` and set `pendingEffectsStatus`.
  - Before mutation: `commitBeforeMutationEffects` (getSnapshotBeforeUpdate => snapshot callbacks).
  - Mutation: `commitMutationEffects` in `ReactFiberCommitWork.js` traverses the effect list and calls host renderer mutation functions (`commitHostUpdate`, `commitPlacement`, `commitDeletion`).
  - Swap the root trees: `root.current = finishedWork`.
  - Layout phase: `commitLayoutEffects` (calls `useLayoutEffect` and class lifecycle methods like componentDidMount/componentDidUpdate).
  - Passive phase: `flushPassiveEffects` (useEffect), usually run asynchronously after commit.

## 4. Concurrency, Suspense and Transitions

- Interrupt & Resume: concurrent rendering via `workLoopConcurrent` uses `shouldYield` to time-slice rendering. Work-in-progress is preserved across yields and resumed later. High-priority updates can preempt lower-priority updates.
- Lanes & Prioritization: Updates are assigned to lanes (Sync, Default, Transition, Retry, Idle) and composed as bitmasks. `getNextLanes` is the key function that determines the next lanes to process for a root.
- Suspense: When a component throws a Promise during render, React marks that fiber as suspended. The render pauses and can be retried when the Promise resolves (ping) or fallback content can be committed after a timeout.
- Transitions: Updates scheduled in `startTransition` use Transition lanes. React may use heuristics like `shouldAttemptEagerTransition` to decide whether to render transitions synchronously in special cases (e.g., popstate).

## 5. End-to-End Flow Diagram (Mermaid)

Below is a mermaid flowchart illustrating the end-to-end rerender flow. All labels are in English.

```mermaid
flowchart TD
  subgraph Event
    E(setState / dispatch / props change)
  end
  E --> S[scheduleUpdateOnFiber(root, fiber, lane)]
  S --> M[markRootUpdated(root, lane)]
  M --> Q[ensureRootIsScheduled(root)]
  Q --> R[ReactFiberRootScheduler / Scheduler]
  R --> P[scheduleCallback -> performWorkOnRoot(root, lanes)]
  P --> choose{shouldTimeSlice?}
  choose -->|yes| RC[renderRootConcurrent]
  choose -->|no| RS[renderRootSync]
  RC & RS --> prep[prepareFreshStack / workLoop]
  prep --> Begin[beginWork -> reconcileChildFibers]
  Begin --> Complete[completeWork -> build effect list]
  Complete --> WorkLoop[workLoop (traverse, may yield)]
  WorkLoop --> Finished[finishedWork]
  Finished --> CommitDecide[finishConcurrentRender]
  CommitDecide -->|commit| Commit[commitRoot root.current = finishedWork]
  Commit --> Before[commitBeforeMutationEffects]
  Before --> Mutation[commitMutationEffects (DOM ops)]
  Mutation --> Layout[commitLayoutEffects (useLayoutEffect / life-cycles)]
  Layout --> Passive[flushPassiveEffects (useEffect)]
  Passive --> Done[Done]
  classDef default fill:#f8f8ff,stroke:#333,stroke-width:1px;
```

## 6. Key Files (Source Reference)

- Core work loop & scheduler:
  - `packages/react-reconciler/src/ReactFiberWorkLoop.js` — core render and commit flow
  - `packages/react-reconciler/src/ReactFiberRootScheduler.js` — root scheduling and microtask queue
  - `packages/react-reconciler/src/ReactFiberLane.js` — lane bitmask and scheduling utilities
- Rendering and reconciliation:
  - `packages/react-reconciler/src/ReactFiberBeginWork.js` — `beginWork` logic and reconciliation
  - `packages/react-reconciler/src/ReactFiberCompleteWork.js` — `completeWork` and DOM host node shape preparation
- Commit and side effects:
  - `packages/react-reconciler/src/ReactFiberCommitWork.js` — core commit-phase effects, host mutation
  - `packages/react-reconciler/src/ReactFiberCommitEffects.js` — effect handling
- Scheduler:
  - `packages/react-reconciler/src/ReactFiberScheduler.js` — host-level scheduler wrapper
  - `packages/react-reconciler/src/Scheduler.js` — scheduler abstraction used by the reconciler

## 7. Tips & Important Implementation Notes

- Effect list design: React's render phase only marks effects (flags). All DOM mutation happens in the commit phase. This separation enables interruption and resumption of the render phase.
- Time-slicing & preemption: `shouldYield` and `shouldYieldForPrerendering` are used to decide whether to yield the CPU in concurrent mode, minimizing main-thread blocking. React attempts to resume WIP work after yields.
- Commit timing: Concurrent mode sometimes delays commit (e.g., Suspense fallbacks or throttled retries). The `finishConcurrentRender` function encapsulates the heuristics for when to commit.
- Error recovery: `renderRootConcurrent` and `renderRootSync` use helper functions such as `handleThrow`, `recoverFromConcurrentError`, and `markRootSuspended` to retry rendering or fall back as necessary.

## 8. FAQ

- Q: Why does React compute the WIP tree in advance instead of updating the DOM immediately?
  - A: To support interruption/resume and priority scheduling, the render phase is pure computation: it collects DOM operations in an effect list and leaves the actual DOM mutation to the commit phase to ensure consistency.

- Q: How do lanes (priorities) affect rerender behavior?
  - A: Lanes assign different priorities to updates. The scheduler uses lanes to determine which updates to render first; higher-priority lanes can preempt lower-priority work.

- Q: How does Suspense interrupt rendering?
  - A: When a component throws a Promise during render, React suspends the fiber and retries it when the Promise resolves (a ping). Suspense also supports timeouts and fallback commits if necessary.

## 9. Conclusion

React's rerender system includes scheduling (`scheduleUpdateOnFiber`) -> scheduler (`ReactFiberRootScheduler`) -> render loop (`renderRoot*`, `beginWork`, `completeWork`) -> commit (`commitRoot`). Key takeaways:

- Lanes represent priority and influence which work runs first.
- The render phase only computes and marks effects; DOM operations are performed during commit.
- Concurrent mode enables interruption, resume, and priority-based preemption for better responsiveness.
- Suspense, transitions, and retry lanes are key UX-oriented features for controlling rendering behavior.

---

If you'd like, I can also:
- Add code snippets with function signatures and specific line references for each major function.
- Convert the mermaid diagram to an SVG/PNG for slides.
- Produce a condensed quick-reference table for the main functions and their responsibilities.

Which would you like next?
# React Rerender Architecture & Flow

React 重新渲染（Rerender）架构与流程说明

> This document summarizes React's rerender architecture, flow, key functions, and implementation details based on the repository source (packages/react-reconciler / packages/react-dom). It is intended for readers who want to deeply understand how React's reconciler schedules, renders, and commits updates.
>
> 本文档基于项目源码（packages/react-reconciler / packages/react-dom 等）对 React rerender（重新渲染）架构、流程、关键函数与实现细节的梳理，适用于想深入理解 React 协调器（reconciler）如何调度、渲染与提交更新的读者。

---

## 1. Overview (High Level) ✅

1. 概览（高层）

React's rerender flow can be split into the following stages:

React 的 rerender 流程可以按下列阶段划分：

- Schedule (Update Dispatch) — Updates start from setState / dispatch / props changes and pass through `scheduleUpdateOnFiber`, `markRootUpdated`, and `ensureRootIsScheduled` to mark roots and lanes as pending.

- 更新调度（Schedule）——从 setState / dispatch / props 更新开始，经过 `scheduleUpdateOnFiber`、`markRootUpdated`、`ensureRootIsScheduled`，把 root 和更新的 lane 标记为待处理。
- Scheduler — `ReactFiberRootScheduler` adds roots that need work to the queue and uses the `scheduler` package (`scheduleCallback`) to schedule render tasks.

- 任务调度（Scheduler）——`ReactFiberRootScheduler` 把需要工作的 root 添加到任务队列，并通过 `scheduler` 包（`scheduleCallback`）安排渲染任务。
- Render — `performWorkOnRoot` -> `renderRootConcurrent` / `renderRootSync`: build the work-in-progress tree (WIP), reconcile children in `beginWork`, and compute the effect list (Placement/Update/Deletion flags) in `completeWork`.

- 渲染（Render）——`performWorkOnRoot` -> `renderRootConcurrent` / `renderRootSync`：构建 work-in-progress 树（WIP），通过 `beginWork`（reconcile 子树）和 `completeWork` 计算 effect list（Placement/Update/Deletion 等 flag）。
- Commit — `commitRoot`: run before-mutation effects (getSnapshotBeforeUpdate), then mutation (DOM updates), followed by layout effects and passive effects (useEffect). Finally, set `root.current` to the newly committed tree.

- 提交（Commit）——`commitRoot`：先 before-mutation（getSnapshotBeforeUpdate），再 mutation（DOM 操作），最后 layout（layout effects）和 passive（useEffect）等阶段，并且把 `root.current` 指向新的已提交树。


## 2. Key Concepts & Data Structures 🔧

2. 关键概念与数据结构

- Fiber — A unit that represents a piece of UI: it contains type, props, state, child, sibling, flags, updateQueue, etc.

- Fiber：表示 UI 上的一个“单元”，包含 type、props、state、child、sibling、flags、updateQueue 等。
- Work-in-progress (WIP) tree — A temporary tree built during the render phase to compute the next committed tree. When completed, the WIP tree becomes `root.current`.

- work-in-progress（WIP）树：在渲染阶段 React 构建的临时树，用于计算下一棵树。完成后，WIP 会切换到 `root.current`。
- Lanes / Priorities — A bitmask used to represent different priority updates (Sync, Default, Transition, Retry, Idle, etc.). `getNextLanes` picks the next lane(s) to work on.

- Lanes（车道）/优先级：位掩码（bitmask）表示不同优先级的更新（Sync, Default, Transition, Retry, Idle 等）。`getNextLanes` 用于选择下一次要处理的 lane(s)。
- Effect List / flags — A fiber's flags mark side effects to be executed (Placement, Update, Deletions, Passive, etc.).

- EffectList / flags：Fiber 的 flags 标记需要执行的副作用（Placement, Update, Deletions, Passive 等）。
- Scheduler — The `scheduler` package is responsible for scheduling tasks with different priorities in the browser (e.g. `scheduleCallback`, `shouldYield`).

- Scheduler：`scheduler` 包负责在浏览器环境中调度优先级不同的任务（例如 `scheduleCallback`、`shouldYield`）。


## 3. Detailed Flow (Sequence) — Function & File Mapping 🧭

3. 详细流程（顺序） — 代码函数关联

下面是典型的渲染路径（同步或并发模式）以及对应的关键函数和实现文件：

### 3.1 Trigger (触发更新)
- Components call setState/useState/dispatch or external events trigger updates.

- 组件调用 setState/useState/dispatch 或外部事件派生更新。
- Class components: `enqueueSetState` -> `scheduleUpdateOnFiber`.

- class 组件：`enqueueSetState` -> `scheduleUpdateOnFiber`。
- Function components: `dispatchAction` -> `scheduleUpdateOnFiber`.

- function 组件：`dispatchAction` -> `scheduleUpdateOnFiber`。
- Source references: `packages/react-reconciler/src/ReactFiberClassComponent.js`, `ReactFiberHooks.js` for the scheduling paths.

- 源码参考：`packages/react-reconciler/src/ReactFiberClassComponent.js`、`ReactFiberHooks.js` 调度路径。

### 3.2 Schedule (调度)
-- `scheduleUpdateOnFiber(root, fiber, lane)` (file: `packages/react-reconciler/src/ReactFiberWorkLoop.js`)
  - `markRootUpdated(root, lane)`: marks updates on `root.pendingLanes`.

  - `markRootUpdated(root, lane)`：把更新标记到 `root.pendingLanes`。
  - `ensureRootIsScheduled(root)`: enqueue the root in `ReactFiberRootScheduler` and ensure a microtask / task is scheduled.

  - `ensureRootIsScheduled(root)`：把 root 加入 `ReactFiberRootScheduler` 的根队列，并保证有 microtask/任务在运行。
- `ReactFiberRootScheduler.js` selects the next lanes to work on using `getNextLanes(root, wipLanes, rootHasPendingCommit)` and uses `Scheduler.scheduleCallback` to run `performWorkOnRoot`.

- `ReactFiberRootScheduler.js` 根据 `getNextLanes(root, wipLanes, rootHasPendingCommit)` 选出下一次要工作的 lanes，并调用 `Scheduler.scheduleCallback` 来运行 `performWorkOnRoot`。

### 3.3 Work Loop / Render Phase (执行)
-- `performWorkOnRoot(root, lanes, forceSync)` (`ReactFiberWorkLoop.js`)
  - Decides whether to use `renderRootConcurrent` (interruptible) or `renderRootSync` (blocking) depending on `shouldTimeSlice`.

  - 根据 `shouldTimeSlice` 决定使用 `renderRootConcurrent`（可中断）还是 `renderRootSync`（不可中断）。

-- Render core: `renderRootConcurrent` / `renderRootSync` -> `prepareFreshStack(root, lanes)` -> work loop (`workLoopConcurrent`, `workLoopSync`).
  - `workLoop*` iterates the WIP tree by loop calling `performUnitOfWork(unitOfWork)`. `performUnitOfWork` invokes `beginWork` to reconcile and `completeUnitOfWork` when no child is present.
  - `beginWork(current, workInProgress, entangledRenderLanes)` (file: `ReactFiberBeginWork.js`) is responsible for calling function/class component render, producing new fibers (`reconcileChildFibers` / `mountChildFibers`) and setting flags (Placement/Update/Deletion).
  - `completeUnitOfWork` and `completeWork` (`ReactFiberCompleteWork.js`) assemble effects and prepare host nodes (create DOM node shapes), but do not mutate the DOM in this phase.

- Interruptions: Concurrent mode lets `workLoopConcurrent` call `shouldYield()` to interrupt rendering so it can be preempted by higher-priority work. React can resume or restart rendering later (sometimes aborting parts of WIP).

### 3.4 Finish / Finalize (完成/收尾)
- When the render is complete or needs to terminate, `renderRootConcurrent` returns an exit status that triggers `finishConcurrentRender`.
- `finishConcurrentRender` decides whether to commit immediately or delay (e.g., Suspense fallback, throttled retries). If commit proceeds, it calls `commitRootWhenReady` / `commitRoot`.

### 3.5 Commit Phase (提交)
-- `commitRoot(root, finishedWork, lanes, ...)` (`ReactFiberWorkLoop.js`)
  - Clear and flush pending passive effects (`flushPendingEffects`).
  - Compute `remainingLanes` and call `markRootFinished`.
  - Store `finishedWork` as `pendingFinishedWork` and set `pendingEffectsStatus` to prepare commit phases.
  - Before Mutation Phase: `commitBeforeMutationEffects` (call `getSnapshotBeforeUpdate`, etc.)
  - Mutation Phase: `commitMutationEffects` (`packages/react-reconciler/src/ReactFiberCommitWork.js`) traverses the effect list and performs DOM operations (`commitHostUpdate`, `commitPlacement`, `commitDeletion`).
    - `commitMutationEffectsOnFiber` 根据不同 `tag`（HostComponent、ClassComponent、FunctionComponent）执行不同操作，并最终调用 host renderer（`react-dom`）提供的 host mutation 接口。
    - 示例文件：`packages/react-reconciler/src/ReactFiberCommitWork.js`。
  - Swap the root tree (assign `root.current = finishedWork`).
  - Layout Phase: `commitLayoutEffects` (call `useLayoutEffect` callbacks and class lifecycle methods like componentDidMount/componentDidUpdate).
  - Passive Phase: `flushPassiveEffects` (`useEffect`, usually run asynchronously after commit).


## 4. Concurrency, Suspense and Transitions 🧩

4. 并发与 Suspense 特性说明

- Interrupt & Resume: `workLoopConcurrent` enables time-sliced render work, and calls `shouldYield` to decide when to yield. After yielding, in-progress work is preserved in WIP and resumed later. Higher priority updates can preempt lower priority work.

- 中断与恢复：`workLoopConcurrent` 允许在一定时间片内执行 render 并通过 `shouldYield` 中断。中断后会把进度保存在 WIP 并在未来恢复。优先级高的更新会 preempt（抢占）低优先级更新。

- Lanes: updates are assigned to lanes (Sync, Default, Transition, Retry, Idle, ...). Lanes represent priorities and help batching and preemption. `getNextLanes` computes the next lanes to work on for a root.

- Lanes：更新被打上 lane（Sync/Default/Transition/Retry 等），用于调度优先级、批处理与抢占。`getNextLanes` 计算 root 的作用在于是下一次需要执行哪些 lanes。

- Suspense: when a component throws a Promise (Suspense), concurrent rendering will suspend rendering of that fiber and wait for a ping (Promise resolution), or it may timeout and commit a fallback. Relevant code includes `workInProgressSuspendedReason`, `markRootSuspended`, `prepareFreshStack`, and `workInProgressRootPingedLanes`.

- Suspense：当组件抛出一个 Promise（Suspense），并发渲染会把进度保留并等待 ping（`then` resolve），或者会在超时后提交 fallback。Suspense 的相关路径包括 `workInProgressSuspendedReason`、`markRootSuspended`、`prepareFreshStack`、`workInProgressRootPingedLanes` 等。

- Transitions: updates scheduled inside `startTransition` use Transition lanes. React uses lanes and heuristics such as `shouldAttemptEagerTransition` to decide how aggressively to render transition updates (e.g. popstate/restore scenarios may be handled synchronously).

- Transitions：startTransition 所在的更新分配 Transition lane。React 会根据 lanes 和 `shouldAttemptEagerTransition` 来决定是否尽可能快速渲染 transition 更新（例如 popstate 场景的同步优先）


## 5. End-to-End Flow Diagram (Mermaid) 🧭

5. 整体流程示意图（Mermaid）

Below is a mermaid flowchart that describes the end-to-end rerender flow. English labels are primary, with Chinese labels in parentheses.

```mermaid
flowchart TD
  subgraph Event
    E(setState / dispatch / props change):::event
  end
  E --> S[scheduleUpdateOnFiber(root, fiber, lane)\n(调度更新)]
  S --> M[markRootUpdated(root, lane)\n(标记 Root 更新)]
  M --> Q[ensureRootIsScheduled(root)\n(确保 Root 已调度)]
  Q --> R(ReactFiberRootScheduler / Scheduler\n(根调度器 / Scheduler))
  R --> P[scheduleCallback -> performWorkOnRoot(root, lanes)\n(调度回调 -> 开始渲染)]
  P --> choose{shouldTimeSlice ?\n(是否时间切片)}
  choose -->|yes| RC[renderRootConcurrent\n(并发渲染)]
  choose -->|no| RS[renderRootSync\n(同步渲染)]
  RC & RS --> prep[prepareFreshStack / workLoop\n(准备 WIP 与遍历)]
  prep --> Begin[beginWork -> reconcileChildFibers\n(beginWork -> 子树对比)]
  Begin --> Complete[completeWork -> create WIP effects\n(completeWork -> 生成 Effect)]
  Complete --> WorkLoop[workLoop (traverse, may yield)\n(工作循环，可能中断)]
  WorkLoop --> Finished[finishedWork\n(渲染完成的树)]
  Finished --> CommitDecide[finishConcurrentRender\n(评估是否 Commit)]
  CommitDecide -->|commit| Commit[commitRoot root.current = finishedWork\n(执行 Commit)]
  Commit --> Before[commitBeforeMutationEffects\n(前变更阶段)]
  Before --> Mutation[commitMutationEffects (DOM ops)\n(变更阶段 - DOM 操作)]
  Mutation --> Layout[commitLayoutEffects (useLayoutEffect / cDM)\n(布局阶段)]
  Layout --> Passive[flushPassiveEffects (useEffect)\n(被动阶段 - useEffect)]
  Passive --> Done[Done\n(完成)]

  classDef default fill:#f8f8ff,stroke:#333,stroke-width:1px;
  classDef event fill:#e0f7fa,stroke:#333,stroke-width:1px;
```


## 6. Key Files (Source Reference) 📚

6. 关键文件索引（源码参考）
-- Core work loop & scheduler:
  - `packages/react-reconciler/src/ReactFiberWorkLoop.js`（render/commit 主要流程）
  - `packages/react-reconciler/src/ReactFiberRootScheduler.js`（根调度逻辑）
  - `packages/react-reconciler/src/ReactFiberLane.js`（lane/优先级、bitmask）
-- Rendering mechanism:
  - `packages/react-reconciler/src/ReactFiberBeginWork.js`（beginWork，reconciliation）
  - `packages/react-reconciler/src/ReactFiberCompleteWork.js`（completeWork）
-- Commit & side effects:
  - `packages/react-reconciler/src/ReactFiberCommitWork.js`（commit phase）
  - `packages/react-reconciler/src/ReactFiberCommitEffects.js`（effect 处理）
-- Scheduler:
  - `packages/react-reconciler/src/ReactFiberScheduler.js`（host-level scheduler wrapper）
  - `packages/react-reconciler/src/Scheduler.js`（scheduler 封装）


## 7. Tips & Important Implementation Notes 💡

7. 扩展阅读 & 重要实现细节（Tips & 优化点）

-- Effect List design:
  - React 在 render 阶段只标记 effect（flags），直到 commit 才执行 DOM 操作；这使得渲染阶段纯粹（纯计算），便于中断与恢复。

-- Time slicing & preemption:
  - `shouldYield` 与 `shouldYieldForPrerendering` 用于并发场景中决定何时中断（yield）渲染任务，避免阻塞主线程。React 会在恢复时尽可能继续 WIP。

-- Choosing commit timing:
  - 并发模式下，React 可能会延迟 commit（例如 Suspense 或 throttled retries），这也是 `finishConcurrentRender` 中逻辑的要点。

-- Error handling & recovery:
  - `renderRootConcurrent` 与 `renderRootSync` 在遇到错误或 Suspense 时，都会通过 `handleThrow`、`recoverFromConcurrentError`、`markRootSuspended` 等机制回退或重试。


## 8. FAQ (Common Questions) ❓

8. 常见问答（FAQ）
-- Q: Why does React compute the WIP tree in advance instead of updating the DOM right away?
  - A: To support interruption/resume and priority scheduling, the render phase is pure calculation: it collects DOM operations as an effect list and leaves DOM mutation to the commit phase to ensure consistency.

  - A: 为了支持中断/恢复及优先级调度，render 阶段只做纯计算，收集需要的 DOM 操作到 effect list，所有 DOM 操作在 commit 阶段统一执行，保证一致性。

-- Q: How do lanes (priorities) affect rerender behavior?
  - A: Lanes assign different priorities to updates, and the scheduler chooses which lanes to process first. Higher priority updates (Sync) can preempt lower priority work (Transition), improving responsiveness.

  - A: lanes 将不同类型的更新分配到不同优先级，scheduler 根据 lanes 选择将当前 root 的哪些更新优先处理。高优先级更新（Sync）能抢占低优先级（Transition），从而提升响应性。

-- Q: How does Suspense interrupt rendering?
  - A: When a component throws a Promise (or uses `use`) during render, React marks the fiber as suspended and stops rendering that branch; when the Promise resolves (ping), React schedules the root and retries the suspended fiber.

  - A: 当组件抛出 Promise（或使用 `use`）时，render 阶段将停止当前 fiber 的后续渲染并标记为 Suspended；一旦 Promise resolve（ping），React 会为这个 root 再次 schedule 并尝试重试该 fiber。


---

## 9. Conclusion ✅

9. 结论

React's rerender system is composed of `scheduleUpdateOnFiber` -> Scheduler -> renderWorkLoop -> begin/completeWork -> commitRoot. Key takeaways:
- Lanes represent priority and ensure high-priority updates are handled first;
- the render phase only computes and marks effects; DOM operations happen in the commit phase;
- concurrent mode enables interruption, resume, and preemption to improve UX and responsiveness;
- Suspense / Transition / Retry are core mechanisms for rendering control and managing UX.


---

如果你想，我可以：
- 把这个 README 转成更图文并茂的幻灯片或更详尽的流程图；
- 在文中加入更多源码的关键代码片段（函数签名与重要分支）并加上行号引用；
- 生成一张简洁的 PNG/SVG 流程图供演示。 

需要我继续补充哪一块？
