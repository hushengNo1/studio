# Repo 状态管理扫描报告（Zustand / Context）

> 目标：扫描整个仓库的状态管理实现与用法，识别一致性、性能、可维护性与内存风险，并给出可执行的改进建议。
>
> 范围：以 `packages/studio-base/src/` 为主（应用主体），结合根目录架构文档对整体数据流做校准。

---

## 结论摘要

- **总体形态**：仓库主要使用 **Zustand（vanilla store）+ React Context/Provider 包装** 来承载“应用级/模块级状态”。布局类状态（`CurrentLayoutContext`）采用 **自定义订阅/selector**，并内置了 selector 稳定性告警。
- **主要风险点**集中在：
  - **Provider 初始化 store 的写法不一致**，其中至少有一处会在每次 render 时多创建一次 store（虽不影响最终 state，但会造成浪费/潜在副作用）。
  - **状态对象引用变化与深比较**在若干热点路径存在（`_.isEqual`、合并大对象、Map/Record 重建），需要确保 selector 粒度与 memo 策略匹配，否则容易出现“无谓 re-render / 无谓订阅 churn”。
  - **调试输出泄漏**：存在 `console.log` 残留，可能污染日志与性能。

---

## 状态管理方案盘点（Inventory）

### Zustand（主要）

已定位到的 store / provider（非穷尽，但覆盖核心）：

- **Workspace UI 状态（带持久化）**
  - Provider：`packages/studio-base/src/providers/WorkspaceContextProvider.tsx`
  - Context + hook：`packages/studio-base/src/context/Workspace/WorkspaceContext.ts`
  - 特点：`zustand/middleware/persist` + `partialize`（opt-in 持久化字段）
- **MessagePipeline（高频数据流、面板订阅分发）**
  - Store：`packages/studio-base/src/components/MessagePipeline/store.ts`
  - Provider + hooks：`packages/studio-base/src/components/MessagePipeline/index.tsx`
  - 特点：内部/外部（`public`）双层 state，细粒度按 subscriberId/topic 分发
- **PanelState（面板 settings tree、默认标题、强制 remount 序列号）**
  - Provider：`packages/studio-base/src/providers/PanelStateContextProvider.tsx`
  - Context + hook：`packages/studio-base/src/context/PanelStateContext.ts`
- **Problems（会话问题列表）**
  - Provider：`packages/studio-base/src/providers/ProblemsContextProvider.tsx`
  - Context + hook：`packages/studio-base/src/context/ProblemsContext.ts`
- **TimelineInteractionState（hover/global bounds 等交互态）**
  - Provider：`packages/studio-base/src/providers/TimelineInteractionStateProvider.tsx`
  - Context + hook：`packages/studio-base/src/context/TimelineInteractionStateContext.tsx`
- **StudioLogsSettings（日志级别与 channel 开关）**
  - Store：`packages/studio-base/src/providers/StudioLogsSettingsProvider/store.ts`

### 自定义 Context 状态（布局类）

- **CurrentLayoutContext**
  - `packages/studio-base/src/context/CurrentLayoutContext/index.ts`
  - 特点：自建 listener 机制 + `useCurrentLayoutSelector`，并用 `selectWithUnstableIdentityWarning` 做 selector 结果稳定性检测。

### Redux / MobX / Recoil / Jotai

- **Redux**：在 `yarn.lock` 中存在 `redux` 依赖，但在本次扫描的应用代码中未发现典型 Redux 落地（`@reduxjs/toolkit` / `react-redux` / `configureStore` 等）。
- **MobX / Recoil / Jotai**：未发现实际使用。

---

## 关键发现（按风险优先级）

### P0（高优先级）Provider 初始化 store 的不一致与额外创建

在 `TimelineInteractionStateProvider` 中：

- 文件：`packages/studio-base/src/providers/TimelineInteractionStateProvider.tsx`
- 现状：`useState(createTimelineInteractionStateStore())`
- 问题：`createTimelineInteractionStateStore()` 会在 **每次 render** 时被调用一次（即使 React 只会在首次 render 采用 `useState` 的初始值，后续 render 仍会执行该表达式并创建“废弃的 store 实例”）。

影响：

- **性能浪费**：每次 render 都会额外 new 一个 store（含闭包/对象分配）。
- **潜在副作用风险**：如果某个 store 初始化未来引入副作用（订阅、读取环境、日志等），这种“多余执行”会变得危险。

建议：

- 统一为惰性初始化：`useState(createTimelineInteractionStateStore)` 或 `useState(() => createTimelineInteractionStateStore())`。
- 在报告末尾的“统一规范建议”里给出全仓统一写法（见下文）。

> 对比：`WorkspaceContextProvider` 与 `PanelStateContextProvider` 等采用了惰性初始化写法（更安全）。

---

### P1（中高优先级）selector 稳定性与引用变化：需要制度化约束

已存在的防护：

- `useCurrentLayoutSelector` 内部使用了：
  - `useShouldNotChangeOften(selector, ...)`：警告 selector 函数频繁变化
  - `selectWithUnstableIdentityWarning(layoutState, selector)`：在 dev 环境检测 selector 对同一输入是否返回不同引用（提示会导致不必要 re-render）
  - 文件：`packages/studio-base/src/context/CurrentLayoutContext/index.ts`
  - 工具函数：`packages/hooks/src/selectWithUnstableIdentityWarning.ts`

仍需关注的点：

- Zustand 的 `useStore(store, selector)` 对 selector 的“引用稳定性”同样敏感：
  - 如果调用方在 render 中创建新 selector（未 `useCallback`），会造成订阅更新/重算频率上升。
  - `MessagePipeline` 的 `useMessagePipeline` 已在内部用 `useCallback((state) => selector(state.public), [selector])` 做了包裹，但这只能解决“外层 selector 变化导致 wrapper 变化”的问题，无法替代调用方在热点组件中保持 selector 稳定的最佳实践。

建议（制度化）：

- 为所有 store hooks（Zustand/CurrentLayout）建立统一的 **selector 编写约束**：
  - selector 必须是稳定函数（组件内用 `useCallback`，或组件外定义常量 selector）
  - selector 返回值尽量是原始值或稳定引用（避免每次返回新对象/新数组）
  - 若必须返回对象，优先拆分为多个 selector 或配合 shallow/自定义 equality（按项目惯例选择）

---

### P1（中高优先级）深比较与大对象合并：热点路径需谨慎

#### CurrentLayout reducer：`_.isEqual` 与 config 合并

- 文件：`packages/studio-base/src/providers/CurrentLayoutProvider/reducers.ts`
- 现状：`savePanelConfigs` 会对 `oldConfig` 与 `newConfig` 做 `_.isEqual`，用于避免无变化时保持引用不变。

权衡：

- 优点：能显著减少“配置未变但引用变了”导致的上游 re-render。
- 风险：在配置对象较大、更新频繁时，`_.isEqual` 可能成为性能热点。

建议：

- 保留该策略的同时，明确“配置更新频率”的边界：对高频操作（拖拽/实时交互）尽量避免触发大范围 config 合并。
- 在产生 config 的上游（如 panel settings action）尽量保持“最小 patch”，减少 newConfig 体积。

#### TimelineInteractionState：`_.isEqual` 用于 hoverValue

- 文件：`packages/studio-base/src/providers/TimelineInteractionStateProvider.tsx`
- 现状：`setHoverValue` 用 `_.isEqual(newValue, store.hoverValue)` 去重。

建议：

- 若 `HoverValue` 结构稳定且字段少，这个成本可接受；但如果 hover 事件非常高频，建议改成更明确的字段比较（例如比较 `type/value/componentId`），以避免 deep equal 的泛化成本。

---

### P2（中优先级）跨层状态写入：通过 listener 直写 Player 的全局变量

在 `MessagePipelineProvider` 中：

- 文件：`packages/studio-base/src/components/MessagePipeline/index.tsx`
- 行为：通过 `CurrentLayoutContext.addLayoutStateListener` 监听 `globalVariables` 变化，并直接调用 `player.setGlobalVariables(...)`，刻意避免 React re-render。

评价：

- 这是一个合理的“绕过 React 更新”的性能优化点（文内也有解释）。
- 风险在于：这种“非 React 状态路径”的写入如果散落在多处，容易出现难以追踪的数据流。

建议：

- 将这类“桥接逻辑”集中到固定层（如 MessagePipeline/PlayerManager），并在文档中明确“谁是全局变量的唯一写入入口”。
- 为 listener 注册/卸载建立一致规范（这里已经做到 cleanup）。

---

### P3（低优先级但应尽快清理）调试输出泄漏

- 文件：`packages/studio-base/src/panels/ThreeDeeRender/renderables/CameraStateSettings.ts`
- 现状：存在 `console.log("action: ", action);`
- 建议：移除或改为受控日志（使用项目 logger，并受 log level 控制）。

---

## 建议的统一规范（可作为后续重构 checklist）

### Provider 中 store 初始化（统一写法）

推荐统一为惰性初始化，避免每次 render 额外创建：

- ✅ `const [store] = useState(createXStore);`
- ✅ `const [store] = useState(() => createXStore(args));`
- ❌ `const [store] = useState(createXStore());`（会在每次 render 执行 create）

### selector 编写规范（Zustand + CurrentLayout 通用）

- **selector 函数必须稳定**：组件内 `useCallback` 或组件外常量函数。
- **selector 返回值避免新引用**：不要在 selector 里 `return { ... }` 或 `return array.map(...)` 这类每次都新建对象/数组的写法。
- **需要组合返回时**：优先拆分多个 selector；或采用 shallow equality（按现有项目模式定）。

### 持久化（persist）边界清晰化

以 `WorkspaceContextProvider` 为例，`partialize` 采用 opt-in 列表是很好的做法：

- 建议为所有持久化 store 明确：
  - **persist key 命名规范**（如 `fox.workspace`）
  - **version/migrate** 必须存在（避免破坏性升级）
  - **partialize 必须是白名单**（避免意外持久化大对象/敏感字段）

---

## 附录：本次扫描涉及的核心文件清单

- `packages/studio-base/src/providers/WorkspaceContextProvider.tsx`
- `packages/studio-base/src/context/Workspace/WorkspaceContext.ts`
- `packages/studio-base/src/components/MessagePipeline/store.ts`
- `packages/studio-base/src/components/MessagePipeline/index.tsx`
- `packages/studio-base/src/providers/CurrentLayoutProvider/reducers.ts`
- `packages/studio-base/src/context/CurrentLayoutContext/index.ts`
- `packages/hooks/src/selectWithUnstableIdentityWarning.ts`
- `packages/studio-base/src/providers/PanelStateContextProvider.tsx`
- `packages/studio-base/src/context/PanelStateContext.ts`
- `packages/studio-base/src/providers/ProblemsContextProvider.tsx`
- `packages/studio-base/src/context/ProblemsContext.ts`
- `packages/studio-base/src/providers/TimelineInteractionStateProvider.tsx`
- `packages/studio-base/src/context/TimelineInteractionStateContext.tsx`
- `packages/studio-base/src/providers/StudioLogsSettingsProvider/store.ts`
- `packages/studio-base/src/panels/ThreeDeeRender/renderables/CameraStateSettings.ts`

