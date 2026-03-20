# Foxglove Studio 项目架构分析

本文档基于仓库当前代码结构与关键入口/配置文件进行梳理，聚焦整体架构分层、核心模块、数据流与构建体系。

## 1. 仓库与工作区结构

该项目是 Yarn Workspaces 的 Monorepo：

- 根目录 `package.json` 定义了 workspaces（`packages/*`, `packages/@types/*`, `web`, `benchmark`）。
- 主要应用与包分布：
  - `web/`: Web 应用入口与构建配置（薄封装层）。
  - `packages/studio-web/`: Web 应用壳层与渲染入口。
  - `packages/studio-base/`: 核心应用逻辑与 UI 组件库。
  - `packages/studio/`: 共享类型定义。
  - 其他支撑包：`hooks`, `theme`, `log`, `message-path`, `mcap-support` 等。

## 2. 应用入口与启动链路

Web 版本的启动链路清晰分层：

1. **入口文件**：`web/src/entrypoint.tsx`  
   仅调用 `@foxglove/studio-web` 的 `main()`。
2. **Web 应用入口**：`packages/studio-web/src/index.tsx`  
   - 处理浏览器兼容性提示
   - 异步加载 `@foxglove/studio-base`
   - 组装 `WebRoot` 与 `StudioApp`
3. **WebRoot**：`packages/studio-web/src/WebRoot.tsx`  
   - 组装默认数据源工厂
   - 提供 `SharedRoot`（上下文/配置入口）
4. **核心应用**：`packages/studio-base/src/StudioApp.tsx`  
   - 组织全局 Provider 栈
   - 渲染主工作区 `Workspace`

整体链路可描述为：

```
web/entrypoint.tsx
  -> studio-web/main()
    -> WebRoot + StudioApp
      -> SharedRoot
        -> Provider 栈
          -> Workspace
```

## 3. 分层架构设计

### 3.1 入口层（App Shell）

由 `web/` 与 `packages/studio-web/` 组成，负责：

- 运行环境检测与兼容性提示
- 资源加载与初始化（字体、i18n、全局 hook）
- 将 Web 平台参数与默认数据源注入核心应用

### 3.2 核心逻辑层（studio-base）

该层是应用的主体，包含：

- **UI 组件**：`components/`
- **面板系统**：`panels/`（3D、Plot、Map 等）
- **数据源与回放**：`dataSources/` + `players/`
- **上下文与状态**：`context/` + `providers/`
- **扩展能力**：`PanelAPI/`
- **服务层**：`services/`（布局迁移、扩展加载等）

### 3.3 类型与基础支撑

`packages/studio/` 提供公共类型定义；`packages/@types/*` 和 `packages/typescript-transformers/` 等用于工程支撑。

## 4. 核心模块与职责

### 4.1 数据源与回放

- `dataSources/`: 各种数据源工厂（ROS1/ROS2、MCAP、WebSocket 等）
- `players/`: 抽象播放接口与具体实现
- `components/PlayerManager`: 管理 Player 生命周期

### 4.1.1 数据源接入的详细分析

数据源接入由 **工厂接口** + **选择与初始化流程** 组成：

**(1) 数据源工厂协议**

`IDataSourceFactory` 定义了统一的接入协议（`context/PlayerSelectionContext.ts`）：

- `type`: 数据源类别（`file`/`connection`/`sample`）
- `displayName`/`iconName`/`description`: UI 展示信息
- `formConfig`: 连接参数表单（字段定义 + 校验）
- `supportedFileTypes`/`supportsMultiFile`: 文件型数据源能力
- `initialize(args) -> Player`: 根据参数构造 Player

这使得不同数据源（ROS bag、MCAP、本地文件、网络连接等）以一致方式挂载到系统中。

**(2) 默认数据源注册点**

Web 端在 `packages/studio-web/src/WebRoot.tsx` 中创建默认数据源工厂数组并注入 `SharedRoot`：

- `Ros1LocalBagDataSourceFactory`
- `Ros2LocalBagDataSourceFactory`
- `FoxgloveWebSocketDataSourceFactory`
- `RosbridgeDataSourceFactory`
- `UlogLocalDataSourceFactory`
- `McapLocalDataSourceFactory`
- `RemoteDataSourceFactory`
- `SampleNuscenesDataSourceFactory`

此外 `WebRoot` 支持通过 `props.dataSources` 覆盖默认列表，形成可插拔扩展点。

**(3) 选择与初始化流程**

`components/PlayerManager.tsx` 负责接入流程：

1. `selectSource(sourceId, args)` 根据 `id/legacyIds` 定位工厂
2. 处理三类数据源：
   - `sample`: 直接初始化
   - `connection`: 使用 `params` 构造连接型 Player
   - `file`: 支持 `File` 或 `FileSystemFileHandle`
3. `initialize()` 返回 `Player` 后，经 `TopicAliasingPlayer` + `wrapPlayer` 进行包装
4. 通过 `MessagePipelineProvider` 将 Player 注入消息管线

**(4) 最近使用与权限处理**

- 文件句柄 (`FileSystemFileHandle`) 会进行权限检测与申请
- 连接型数据源会记录到 IndexedDB Recents，便于快速回连

总体上，数据源接入是“工厂 + 选择器 + Player 管线”的组合，保证了扩展能力和统一的数据流入口。

### 4.1.2 MCAP 数据源内部实现路径

MCAP 数据源的实现主要分为 **工厂层** 与 **Iterable Source 层**：

**(1) 工厂入口**

- `packages/studio-base/src/dataSources/McapLocalDataSourceFactory.ts`
  - 实现 `IDataSourceFactory`
  - 仅支持 `.mcap` 文件
  - 创建 `WorkerRawIterableSource`，启动 `McapIterableSourceWorker`
  - 通过 `IterablePlayer` 挂载到统一播放管线

**(2) 解析入口（Iterable Source）**

- `packages/studio-base/src/players/IterablePlayer/Mcap/McapIterableSource.ts`
  - 根据输入选择 **Indexed** 或 **Unindexed** 实现
  - 本地文件：优先用 `McapIndexedReader`，失败则降级为 `McapUnindexedIterableSource`
  - 远程 URL：优先 Indexed；无索引时用流式读取

**(3) Indexed 路径**

- `packages/studio-base/src/players/IterablePlayer/Mcap/McapIndexedIterableSource.ts`
  - 使用 `@mcap/core` 的 `McapIndexedReader`
  - 初始化阶段解析 topics/schema、统计信息与时间范围
  - `messageIterator` 通过索引区间读取，支持高效随机访问
  - 支持 `getBackfillMessages`（按 topic 获取指定时间点前的最近消息）

**(4) Unindexed 路径**

- `packages/studio-base/src/players/IterablePlayer/Mcap/McapUnindexedIterableSource.ts`
  - 使用 `McapStreamReader` 流式解析
  - 将全部消息读入内存（限制：< 1GB）
  - 初始化阶段构建 topics、datatypes、统计与问题列表
  - `messageIterator` 在内存中筛选/排序

**(5) Worker 入口**

- `packages/studio-base/src/players/IterablePlayer/Mcap/McapIterableSourceWorker.worker.ts`
  - 作为 `WorkerRawIterableSource` 的执行载体
  - 负责在 Web Worker 中运行 MCAP 解析，避免阻塞主线程

整体链路可表示为：

```
McapLocalDataSourceFactory
  -> WorkerRawIterableSource (Web Worker)
    -> McapIterableSource
      -> McapIndexedIterableSource | McapUnindexedIterableSource
        -> IterablePlayer -> MessagePipeline
```

### 4.2 Panel 系统与注册、加载流程

`panels/` 目录集中实现面板渲染逻辑。面板的可用列表由 **PanelCatalog** 提供，布局中每个格子通过 **panelId**（如 `3D!xxx`）对应一个面板类型与一份配置；面板模块按需懒加载，并通过 **Panel** HOC 注入 config/saveConfig，扩展类面板再经 **PanelExtensionAdapter** 调用 `initPanel` 完成挂载。下面分注册、加载、以 3D 为例三部分说明。

#### 4.2.1 面板注册

**(1) 内置面板列表（PanelInfo）**

- 定义位置：`packages/studio-base/src/panels/index.ts`
- `getBuiltin(t: TFunction)` 返回 `PanelInfo[]`，每个元素包含：
  - `type`：面板类型字符串，唯一标识（如 `"3D"`、`"Image"`、`"Plot"`）
  - `title`、`description`、`thumbnail`：用于面板选择器与布局 UI
  - `module`：**懒加载函数** `() => Promise<{ default: PanelComponent }>`，例如 3D 为 `async () => await import("./ThreeDeeRender")`
- 内置面板不直接导出组件，而是通过 `module()` 在需要时才加载对应 chunk，减少首屏体积。

**(2) PanelCatalog 的提供**

- 提供者：`packages/studio-base/src/providers/PanelCatalogProvider.tsx`
- 在 `StudioApp` 中置于 `PanelCatalogProvider` 下（见 `StudioApp.tsx`），子组件可通过 `usePanelCatalog()` 使用。
- 行为：
  - 使用 `panels.getBuiltin(t)` 得到内置面板列表（随 i18n 的 `t` 更新）
  - 合并扩展面板（`ExtensionCatalog` 的 `installedPanels`）、`useAppContext().extraPanels`
  - 按 `type` 建 `panelsByType` Map，对外提供：
    - `getPanels()`：返回全部可用面板信息
    - `getPanelByType(type)`：根据类型取单个 `PanelInfo`（含 `module`）

**(3) 布局中的面板 ID 与类型**

- 布局由 `CurrentLayoutProvider` 管理，保存为 Mosaic 树 + `configById`（panelId -> 面板配置）。
- 面板 ID 形如 `3D!1a2b3c`，通过 `getPanelTypeFromId(id)` 得到类型 `"3D"`，用于在 Catalog 中查找 `PanelInfo` 并加载 `module`。

#### 4.2.2 面板加载与渲染

**(1) 布局渲染入口**

- `Workspace` 内使用 `CurrentLayoutProvider` 的 layout 状态，渲染 `PanelLayout`。
- `PanelLayout`（`packages/studio-base/src/components/PanelLayout.tsx`）根据当前 `layout`（Mosaic 树）递归渲染每个 tile。

**(2) 懒加载与 Tile 渲染**

- 在 `UnconnectedPanelLayout` 中：
  - `panelCatalog.getPanels()` 得到所有 `PanelInfo`
  - `panelComponents = new Map(panelInfo.type -> React.lazy(panelInfo.module))`，即每个类型对应一个 `React.lazy(() => module())`，**首次渲染该类型时才会执行 `module()` 加载 chunk**
  - 对每个 tile，`id` 为 panelId，`type = getPanelTypeFromId(id)`，取 `PanelComponent = panelComponents.get(type)`，若存在则渲染 `<PanelComponent childId={id} tabId={tabId} />`，否则渲染 `UnknownPanel`
  - 外层用 `<Suspense>` 包住，加载中显示 loading。

**(3) Panel HOC 与 config 注入**

- 内置面板的 `module()` 返回的 default 多为 `Panel(SomeAdapter, { panelType, defaultConfig })` 包装后的组件（见 `Panel.tsx`）。
- `Panel` 是 HOC：接收 `PanelComponent`（如 ThreeDeeRenderAdapter），返回的 `ConnectedPanel` 接收 `childId`、`tabId`、可选的 `overrideConfig`。
  - 通过 `useConfigById(childId)` 从 `CurrentLayoutContext` 的 `selectedLayout.data.configById[childId]` 读取 `savedConfig`，以及 `savePanelConfigs` 封装出的 `saveConfig`
  - 若没有保存过配置或 defaultConfig 有新字段，会用 `saveConfig` 写回默认/合并配置
  - `panelComponentConfig = { ...defaultConfig, ...savedConfig, ...overrideConfig }`，和 `saveConfig` 一起作为 props 传给真正的面板组件：`<PanelComponent config={panelComponentConfig} saveConfig={saveConfig} ... />`
- 同时 `Panel` 提供 `PanelContext`（含 id、title、config、saveConfig、openSiblingPanel、replacePanel 等），供工具栏、设置等使用。

**(4) 扩展类面板与 PanelExtensionAdapter**

- 使用 `PanelExtensionAdapter` 的面板（如 3D、Image）不直接渲染 React 子树，而是：
  - 在 `useLayoutEffect` 中创建一个 `panelElement`（div），挂到 `panelContainerRef.current`
  - 调用 `initPanel({ panelElement, ...partialExtensionContext, set onRender(fn) })`；`partialExtensionContext` 包含 `initialState`（即 config）、`saveState`（即 saveConfig）、`layout`、`seekPlayback`、`watch`、`subscribe`、`setVariable`、`setParameter`、`unstable_fetchAsset` 等
  - 面板实现方在 `initPanel` 里把 React 或任意 UI 挂到 `panelElement`，并设置 `context.onRender`，在每帧由 Adapter 调用并传入 `RenderState`（currentTime、topics、currentFrame、allFrames、didSeek 等）
  - Adapter 根据面板的 `watch()` 订阅字段和 `subscribe()` 的订阅列表，从 MessagePipeline 取数据，通过 `buildRenderState` 生成 `RenderState`，在 `useLayoutEffect` 中调用 `renderFn(renderState, done)`，面板在 `done()` 后 Adapter 会 `resumeFrame`，完成与播放器的帧同步
  - 卸载时执行 `initPanel` 返回的 cleanup，并移除 `panelElement`、清空该 panelId 的订阅与发布

#### 4.2.3 以 3D 面板为例的完整链路

**(1) 注册**

- `panels/index.ts` 中 3D 面板：`type: "3D"`，`module: async () => await import("./ThreeDeeRender")`。
- Image 面板：`type: "Image"`，`module: async () => await import("./Image")`；`Image/index.tsx` 仅 re-export `ImagePanel`，而 `ImagePanel` 来自 `ThreeDeeRender/index.tsx`，即同一套实现、`interfaceMode: "image"`。

**(2) 用户添加 3D 面板**

- 用户从面板选择器选「3D」→ 调用 `addPanel({ id: getPanelIdForType("3D"), config })`，布局中新增一个节点 id（如 `3D!abc`），`configById["3D!abc"]` 为该面板配置（可先为空，由 Panel 的 useLayoutEffect 写默认 config）。

**(3) 首次渲染该 tile**

- `PanelLayout.renderTile(id="3D!abc")` → `type = "3D"`，`PanelComponent = panelComponents.get("3D")`，即 `React.lazy(() => import("./ThreeDeeRender"))`。
- React 渲染 `<PanelComponent childId="3D!abc" tabId={...} />`，触发 lazy 加载，加载完成后得到 `Panel(ThreeDeeRenderAdapter.bind(undefined, "3d"), { panelType: "3D", defaultConfig: {} })`。

**(4) Panel 层**

- `ConnectedPanel` 用 `childId="3D!abc"` 调用 `useConfigById`，得到 `config`（来自 layout 的 configById）和 `saveConfig`；合并 defaultConfig 后得到 `panelComponentConfig`，传给 `<ThreeDeeRenderAdapter config={...} saveConfig={...} />`。

**(5) ThreeDeeRenderAdapter 与 initPanel**

- `ThreeDeeRender/index.tsx` 中 `ThreeDeeRenderAdapter(interfaceMode, props)` 从 props 解出 `config`、`saveConfig`（即 Panel 注入的），通过 `PanelExtensionAdapter` 的 `initPanel` 传入的实为 `initialState`/`saveState`。
- `boundInitPanel` 在 Adapter 的 `useLayoutEffect` 中被调用：`initPanel({ panelElement, initialState, saveState, watch, subscribe, onRender, ... })`。
- 实际执行的是 `initPanel`（即 `initPanel` 函数）：在 `panelElement` 上 `ReactDOM.render(<ThreeDeeRender context={context} interfaceMode="3d" ... />)`，其中 `context` 是 Adapter 构造的 `BuiltinPanelExtensionContext`（含 `initialState`、`saveState`、`watch`、`subscribe`、`onRender` 等）。

**(6) ThreeDeeRender 组件内**

- `ThreeDeeRender.tsx` 用 `useState`/`useEffect` 管理 config、canvas ref、renderer 实例；在 `useEffect` 中根据是否有 canvas 创建 `new Renderer({ canvas, config, sceneExtensionConfig, ... })`，并同步 config、topics、currentTime、didSeek 等。
- 通过 `context.watch("currentTime"|"allFrames"|"topics"|...)` 声明依赖，Adapter 在每帧根据 `buildRenderState` 把 `currentFrame`、`allFrames`、`currentTime` 等传入 `context.onRender(renderState, done)`；ThreeDeeRender 在 onRender 回调里 setState，进而 effect 更新 renderer 的 currentTime、调用 `handleAllFramesMessages`、`queueAnimationFrame` 等，驱动 3D 渲染循环。
- 订阅列表由 Renderer 根据各 SceneExtension 的 `getSubscriptions()` 与 config 中的 visible 等汇总为 `topicsToSubscribe`，通过 `context.subscribe(topicsToSubscribe)` 传给 MessagePipeline，消息经 pipeline 进入 `renderState.currentFrame`/`allFrames`，再经 `addMessageEvent` 进入 Renderer 的队列，在下一帧 `#handleSubscriptionQueues` 中分发给各扩展。

**(7) 小结（3D 面板从注册到一帧渲染）**

```
getBuiltin() 注册 type "3D" + module()
  -> PanelCatalogProvider 提供 getPanelByType("3D")
  -> 布局中添加 3D 面板 -> configById["3D!xxx"]
  -> PanelLayout 渲染 tile -> React.lazy(ThreeDeeRender module) -> Panel(ThreeDeeRenderAdapter)
  -> Panel 从 useConfigById(childId) 取 config/saveConfig 传入 Adapter
  -> PanelExtensionAdapter 创建 panelElement，initPanel(panelElement, context)
  -> initPanel 内 ReactDOM.render(<ThreeDeeRender context={context} />)
  -> ThreeDeeRender 挂 canvas、创建 Renderer、context.watch/subscribe、context.onRender 里更新 currentTime/allFrames 并驱动 Renderer
  -> Renderer 每帧 #frameHandler：处理消息队列、startFrame(updatePose)、gl.render
```

### 4.3 Provider 栈与上下文

`StudioApp` 中定义了核心 Provider 组合，主要包含：

- 问题/日志/通知上下文
- 播放管理与时间线交互
- 布局管理与面板注册

这些 Provider 形成共享运行时环境，是面板与 Workspace 的依赖基础。

## 5. 构建与开发体系

### 5.1 Web 构建

- `web/webpack.config.ts` 使用 `@foxglove/studio-web` 中的 `webpackConfigs`
- `packages/studio-web/src/webpackConfigs.ts` 复用 `studio-base` 的 `makeConfig`

构建链路：

```
web/webpack.config.ts
  -> studio-web/webpackConfigs
    -> studio-base/webpack.makeConfig
```

### 5.2 TypeScript 工程

根 `tsconfig.json` 使用 project references，支持增量编译与多包协作。

### 5.3 测试与工具

根 `package.json` 包含：

- Jest 测试（含 web/integration-test）
- ESLint + 自定义规则（`packages/eslint-plugin-studio`）
- Storybook（`.storybook/`）

## 6. 典型数据流简述

1. **数据源加载**（`WebRoot` 创建默认数据源工厂）
2. **播放器管理**（`PlayerManager` 维护 Player 生命周期）
3. **消息分发与面板订阅**（Panel API + Panel Catalog）
4. **Workspace 渲染**（布局/面板状态驱动 UI）

## 7. 扩展点与可插拔能力

- **Panel API**：`PanelAPI/` 提供面板扩展机制
- **数据源工厂**：`IDataSourceFactory` 支持自定义数据接入
- **Provider 注入**：`WebRoot` 支持 `extraProviders` 注入

## 8. 3D 渲染架构（ThreeDeeRender）

3D 面板是 Foxglove Studio 中用于可视化机器人/传感器数据的核心面板，基于 **Three.js + WebGL**，采用「Renderer + SceneExtension + Renderable」的可扩展架构。

### 8.1 整体分层

```
Panel 注册 (index.tsx)
  -> PanelExtensionAdapter
    -> ThreeDeeRender (React)
      -> context.onRender / context.subscribe / context.watch
      -> <canvas> ref -> Renderer (纯 TS，无 React)
        -> #scene (THREE.Scene)
          -> SceneExtension (THREE.Object3D)
            -> Renderable (THREE.Object3D + userData)
```

- **React 层**：`ThreeDeeRender.tsx` 负责 canvas 挂载、config 持久化、与 `PanelExtensionContext` 的对接（订阅、currentTime、allFrames、settings 等），不直接持有 Three 对象。
- **渲染核心**：`Renderer` 类持有 `THREE.Scene`、`THREE.WebGLRenderer`、扩展表、订阅表、TransformTree 等，与 React 通过 ref/effect 同步 config、topics、currentTime，通过 `requestAnimationFrame` 驱动每帧渲染。

### 8.2 入口与面板形态

- **入口**：`packages/studio-base/src/panels/ThreeDeeRender/index.tsx`
  - 默认导出 3D 面板：`Panel(ThreeDeeRenderAdapter.bind(undefined, "3d"), { panelType: "3D", ... })`
  - 另导出 **Image 面板**：同一套 `ThreeDeeRender`，但 `interfaceMode: "image"`，仅启用 ImageMode 相关扩展与“仅图像”订阅模式。
- **初始化**：`initPanel` 在 `context.panelElement` 上挂载 `ThreeDeeRender`；可注入 `customSceneExtensions` 覆盖/扩展默认场景扩展。

### 8.3 Renderer 核心职责

`Renderer`（`Renderer.ts`）在绑定到 `<canvas>` 时创建，主要职责包括：

- **场景与光照**：创建 `THREE.Scene`，添加定向光、半球光、坐标轴辅助线；管理背景色、阴影等。
- **扩展管理**：根据 `SceneExtensionConfig`（默认见 `SceneExtensionConfig.ts`）初始化 **reserved**（ImageMode、MeasurementTool、PublishClickTool）与 **extensionsById**（Markers、PointClouds、Poses、FrameAxes、Grids、Urdfs 等），全部以 `SceneExtension` 形式加入 `sceneExtensions` 并挂到 `#scene`。
- **订阅表**：维护 `topicSubscriptions`（topicName -> handlers）与 `schemaSubscriptions`（schemaName -> handlers）；扩展通过 `getSubscriptions()` 声明按 topic 或按 schema 的订阅，由 Renderer 统一注册并驱动 `context.subscribe(topicsToSubscribe)`。
- **坐标变换**：内建 TF 订阅（foxglove.FrameTransform、FrameTransforms、tf2_msgs/TFMessage、geometry_msgs/TransformStamped），将消息写入 `TransformTree`；提供 `addCoordinateFrame`、`normalizeFrameId`（ROS 下去前导 `/`）等。
- **消息入队**：`addMessageEvent(messageEvent)` 从消息中抽取 frame_id 并补充坐标帧，再根据 topic 与 schemaName 将消息推入对应 subscription 的 `queue`；preload 路径下由 `handleAllFramesMessages(allFrames)` 按 currentTime 推进 allFrames 游标并批量 `addMessageEvent`。
- **每帧逻辑**：`#frameHandler(currentTime)` 内依次：处理各订阅队列（`#handleSubscriptionQueues`）、更新固定帧、清屏、发出 `startFrame`、各扩展 `startFrame(...)`（其中会对接 TransformTree 做 `updatePose`）、主场景渲染；若有选中对象则再渲染选中高亮层。
- **动画调度**：`animationFrame()` 调用 `#frameHandler` 后通过 `queueAnimationFrame()` 再次 `requestAnimationFrame(animationFrame)`，形成循环；外部通过 `queueAnimationFrame()` 触发重绘。

### 8.4 SceneExtension 与 Renderable

- **SceneExtension**（`SceneExtension.ts`）：
  - 继承 `THREE.Object3D`，作为场景子节点，位于渲染坐标系原点（render frame origin）。
  - 通过 `getSubscriptions()` 返回 `AnyRendererSubscription[]`（topic 或 schema），由 Renderer 汇总后参与 `context.subscribe` 与消息入队。
  - 维护 `renderables: Map<string, Renderable>`；在 `startFrame(currentTime, renderFrameId, fixedFrameId)` 中根据 `transformTree` 对每个 Renderable 调用 `updatePose(...)`，将其在自身 frame 下的 pose 变换到 render frame，并写回 `position`/`quaternion`；同时根据用户设置更新 `visible` 与 settings 错误（如缺失变换）。
  - 可覆盖 `settingsNodes()`、`handleSettingsAction`、`saveSetting` 以接入右侧设置树；支持 `setColorScheme`、拖放路径等。

- **Renderable**（`Renderable.ts`）：
  - 继承 `THREE.Object3D`，带有 `userData: BaseUserData`（如 `frameId`、`pose`、`messageTime`、`receiveTime`、`settings`、`settingsPath` 等），供 `updatePose` 与设置树使用。
  - 可选 `pickable`、`pickableInstances`、`details()`、`instanceDetails(instanceId)`，用于 Picker 与对象详情面板。

典型数据流：消息经 Renderer 入队 → 每帧 `#handleSubscriptionQueues` 调用各 subscription 的 `handler` → 扩展在 handler 中创建/更新 Renderable 并放入 `renderables` 与 `this.add(renderable)` → 下一帧 `startFrame` 中统一 `updatePose` 再渲染。

### 8.5 坐标变换体系

- **TransformTree**（`transforms/TransformTree.ts`）：
  - 管理多个 **CoordinateFrame**，每个 frame 可存一段时间的 transform 历史（含容量与时间窗口限制），用于按时间插值。
  - `addTransform(childFrameId, parentFrameId, time, transform)` 建立父子关系并写入历史；检测环路（CYCLE_DETECTED）并上报错误。
  - `apply(outPose, srcPose, renderFrameId, fixedFrameId, srcFrameId, dstTime, srcTime)` 将 `srcFrameId` 下在 `srcTime` 的 pose 变换到 `renderFrameId` 在 `dstTime` 下的坐标系。

- **updatePose**（`updatePose.ts`）：根据 Renderable 的 `userData.pose`、`frameId`、`messageTime` 与是否 frameLocked，调用 `transformTree.apply` 得到在 render frame 下的位姿，写入 `Object3D.position` 和 `quaternion`，并据此设置 `visible`。

- **固定帧与跟随**：Renderer 根据 `config.followTf` 解析出 `fixedFrameId`（render frame 所在树的根）；相机可由 `ICameraHandler` 与 follow 模式绑定到某 frame。

### 8.6 消息流与订阅

- **订阅来源**：由 Renderer 聚合所有 SceneExtension 的 `getSubscriptions()`，再结合 `config.topics[t].visible`、`config.imageMode.annotations`、`config.layers` 等得到 `topicsToSubscribe`，通过 `context.subscribe(topicsToSubscribe)` 与消息管线对接；支持按 schema 的 `convertTo`（如将某 topic 转为另一种 schema）订阅。
- **消息注入**：
  - 实时：`context.onRender` 收到 `currentFrame` 后，在 React 侧 setState，effect 中把 `currentTime`、`allFrames`、`didSeek` 等传给 Renderer；Renderer 在 `handleAllFramesMessages(allFrames)` 中按 currentTime 推进游标并调用 `addMessageEvent`；此外若有按帧的 currentFrame 注入路径，也会进入 `addMessageEvent`。
  - `addMessageEvent` 仅做 frame_id 抽取与入队（`queueMessage`），不直接执行 handler；handler 在每帧 `#handleSubscriptionQueues` 中执行，便于批量更新与与渲染同步。
- **队列与过滤**：每个 `RendererSubscription` 可有 `queue`、`filterQueue`（如 `onlyLastByTopicMessage`）；每帧处理完即清空 queue，避免重复处理。

### 8.7 相机与交互

- **ICameraHandler**（`renderables/ICameraHandler.ts`）：扩展形式的相机控制器，实现 `getActiveCamera()`、`setCameraState`、`getCameraState`、`handleResize`；3D 模式常用实现为 `CameraStateSettings` 等，与 `config.cameraState`、`config.followTf`、`config.followMode` 联动。
- **Input**（`Input.ts`）：封装 canvas 上的 resize、click 等，并驱动 `#resizeHandler`、`#clickHandler`。
- **Picker**（`Picker.ts`）：基于 GPU 的拾取——在点击处用小视口做一次用 objectId 着色的离屏渲染，读回像素得到被点击的 Renderable 与可选的 instanceIndex，用于选中高亮与「Object Details」变量（如 `selected_id`）。

### 8.8 配置与设置树

- **RendererConfig**（`IRenderer.ts`）：包含 `cameraState`、`followTf`、`followMode`、`scene`（背景、统计、transform 显示、syncCamera 等）、`transforms`、`topics`、`layers`、`publish`、`imageMode` 等；由 React 侧 `initialState`/`saveState` 持久化，并通过 `renderer.config = config` 与 `renderer.updateConfig()` 与 Renderer 同步。
- **SettingsManager**：维护一棵动态的 settings tree，供右侧面板渲染；各扩展通过 `settingsNodes()` 与 `handleSettingsAction` 注册节点与动作，错误通过 `settings.errors` 挂到对应 path（如缺失变换、TF 溢出）。

### 8.9 Image 模式与 ImageOnly 订阅

- **interfaceMode: "image"**：同一 `ThreeDeeRender`，但只加载 ImageMode、MeasurementTool 等与图像相关的扩展；若未选 calibration，可调用 `enableImageOnlySubscriptionMode()`，关闭所有非 ImageMode 扩展的订阅与 TF 订阅，仅保留图像相关 topic，避免在无相机标定下请求 3D 数据。
- **disableImageOnlySubscriptionMode()**：恢复完整 3D 订阅与 TF 订阅。

### 8.10 3D 渲染数据流小结

1. **配置与时间**：React 从 context 得到 topics、currentTime、allFrames、didSeek 等，写入 Renderer 并触发 seek/clear 逻辑。
2. **订阅**：Renderer 根据扩展订阅与 config 生成 `topicsToSubscribe` → `context.subscribe`；TF 类 schema 由 Renderer 内建订阅。
3. **消息入队**：currentFrame / allFrames 消息经 `addMessageEvent` 按 topic 与 schema 入队，并补充 coordinate frame。
4. **每帧**：`#frameHandler` → 处理队列（执行各 handler）→ `startFrame`（updatePose 等）→ 主渲染 → 选中层渲染 → `endFrame`。
5. **扩展**：各 SceneExtension 通过 handler 更新 Renderable，通过 `startFrame` 与 TransformTree 对齐到同一渲染坐标系，实现多数据源、多类型的统一 3D 展示。

---

## 9. 总结

Foxglove Studio 采用清晰的分层架构：

- **入口层**薄而稳定，处理平台差异与初始化
- **核心层**集中所有 UI、数据与交互逻辑
- **支撑层**提供类型、工具与工程能力
- **3D 渲染**在核心层内采用 Renderer + SceneExtension + Renderable + TransformTree 的扩展式架构，与消息管线、配置和设置树紧密集成

这种结构利于扩展、复用和跨平台移植，是复杂前端应用常见且可维护性较高的组织方式。
