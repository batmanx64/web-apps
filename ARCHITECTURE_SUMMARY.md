# web-apps 项目架构速览

该仓库是 ONLYOFFICE 编辑器前端层，主要负责文档、表格、演示文稿、PDF/表单等编辑器的浏览器端界面与交互。

## 项目定位

- 前端 UI 层：面向用户的编辑界面（工具栏、侧栏、状态栏、对话框等）。
- 宿主通信层：通过 `postMessage` 与外部集成平台通信（如初始化、打开文档、权限同步、导出等事件）。
- SDK 对接层：与 `sdkjs` 引擎配合实现实际编辑能力。

## 目录与职责

- `apps/`：业务主代码（各编辑器 + 通用模块 + API）。
  - `apps/documenteditor|spreadsheeteditor|presentationeditor|pdfeditor|visioeditor/`：按编辑器拆分。
  - `apps/common/`：跨编辑器复用模块（核心框架、组件、控制器、工具函数、资源）。
  - `apps/api/`：对外集成 API 代码。
- `build/`：Grunt 构建系统和打包配置（按编辑器拆分配置文件）。
- `test/`：单测与示例测试页面。
- `translation/`：多语言资源与处理脚本。
- `vendor/`：第三方依赖（RequireJS、Backbone、jQuery、Underscore 等）。

## 启动链路（main 编辑器）

1. `apps/*/main/index.html` 通过 `<script data-main="app_dev" src=".../require.js">` 启动 RequireJS。
2. `apps/*/main/app_dev.js` 进行 `require.config`，配置路径与 shim。
3. 加载 `core` 后创建 `Backbone.Application` 实例，声明控制器列表。
4. `Common.Locale.apply(...)` 完成本地化注入后再异步加载控制器/视图模块并 `app.start()`。
5. `apps/common/main/lib/core/application.js` 完成控制器实例化、事件总线连接与生命周期启动。

## 技术栈

- 模块系统：RequireJS（AMD）
- 框架：Backbone + Underscore + jQuery
- 构建：Grunt（含 requirejs/less/uglify/terser/htmlmin/cssmin/imagemin 等）
- 样式：Less + CSS
- 通信：`window.postMessage`（Gateway）
- 国际化：按语言 JSON 动态加载（`fetch('locale/<lang>.json')`）

## 核心入口与关键模块

- 页面入口：
  - `apps/documenteditor/main/index.html`
  - `apps/spreadsheeteditor/main/index.html`
  - `apps/presentationeditor/main/index.html`
  - `apps/pdfeditor/main/index.html`
- 应用入口：
  - `apps/documenteditor/main/app_dev.js`
  - `apps/spreadsheeteditor/main/app_dev.js`
  - `apps/presentationeditor/main/app_dev.js`
  - `apps/pdfeditor/main/app_dev.js`
- 核心框架：`apps/common/main/lib/core/application.js`
- 宿主网关：`apps/common/Gateway.js`
- 国际化：`apps/common/locale.js`
- 对外 API：`apps/api/documents/api.js`

## 如何启动开发调试

> 说明：本仓库前端运行依赖同级目录的 `sdkjs` 仓库（路径为 `../../../../sdkjs/...`），`index.html` 会直接引用 `sdkjs/develop/sdkjs/*/scripts.js`。

### 1) 准备依赖

```bash
cd build
./sprites.sh
npm install
```

### 2) 启动静态服务并打开调试页面

在仓库根目录执行任意静态服务器（示例用 Python）：

```bash
cd /path/to/web-apps
python3 -m http.server 8080
```

浏览器访问（按需选择）：

- `http://localhost:8080/apps/documenteditor/main/index.html`
- `http://localhost:8080/apps/spreadsheeteditor/main/index.html`
- `http://localhost:8080/apps/presentationeditor/main/index.html`
- `http://localhost:8080/apps/pdfeditor/main/index.html`

调试建议：
- 通过浏览器 DevTools 观察 Network，确认 `sdkjs/develop/sdkjs/.../scripts.js`、`app_dev.js`、locale json 正常加载。
- 在 `apps/*/main/app_dev.js` 的 controller 列表附近打断点，确认控制器生命周期是否按预期执行。

## 如何编译（打包）

### 1) 全量编译（与 CI 一致）

```bash
cd build
./sprites.sh
npm install
grunt
```

默认会执行 common + 各编辑器组件的部署任务。

### 2) 按编辑器编译

```bash
cd build
grunt deploy-documenteditor
grunt deploy-spreadsheeteditor
grunt deploy-presentationeditor
grunt deploy-pdfeditor
grunt deploy-visioeditor
```

### 3) 产物目录

编译产物输出到 `deploy/web-apps/` 下，例如文档编辑器主端为：

- `deploy/web-apps/apps/documenteditor/main/app.js`
- `deploy/web-apps/apps/documenteditor/main/code.js`
- `deploy/web-apps/apps/documenteditor/main/resources/...`


## 外部服务代码位置与调用方式

### 1) 宿主平台（Portal / 父页面）通信

- 代码位置：`apps/common/Gateway.js`、`apps/api/documents/api.js`
- 调用方式：通过 `window.postMessage` 双向通信。
  - 编辑器 -> 宿主：发送 `onAppReady`、`onRequestUsers`、`onRequestClose` 等事件。
  - 宿主 -> 编辑器：发送 `init/openDocument/...` 命令，`Gateway` 中按 `commandMap` 分发。

### 2) 文档服务回调（callbackUrl）

- 代码位置：各编辑器 `Main` 控制器（例如 `apps/pdfeditor/main/app/controller/Main.js`）
- 调用方式：将 `editorConfig.callbackUrl` 注入 `Asc.asc_CDocInfo`，后续由 SDK 引擎（`sdkjs`）在保存、协作等流程中访问后端回调地址。

### 3) 本地化资源请求

- 代码位置：`apps/common/locale.js`
- 调用方式：前端使用 `fetch('locale/<lang>.json')` 动态拉取语言包，失败时回退默认语言。

### 4) 其他网络探测/请求点

- 代码位置：`apps/common/checkExtendedPDF.js`
- 调用方式：使用 `XMLHttpRequest` + `Range` 进行部分内容下载探测（用于 PDF 扩展能力判断流程）。

### 5) 实时协作相关

- 代码位置：`apps/*/main/app_dev.js` / `app.js`（RequireJS 路径里有 `socketio`），以及页面中加载 `sdkjs/develop/sdkjs/*/scripts.js`。
- 调用方式：前端壳层加载 socketio 与 sdkjs；具体实时协作网络连接由 sdkjs 引擎建立与管理。


## License/并发限制相关逻辑位置

结论：**有**。前端存在基于 license 结果的并发/编辑能力限制处理，主要在各编辑器 `Main` 控制器中。

### 关键位置（以文档编辑器为例）

1. 注册 license 变化回调并拉取权限
   - `asc_onLicenseChanged` 回调注册
   - `asc_getEditorPermissions(licenseUrl, customerId)` 拉取授权信息

2. 根据 license 类型识别并发限制
   - 会识别 `Connections / UsersCount / ConnectionsOS / UsersCountOS / ConnectionsLive / UsersViewCount / SuccessLimit` 等结果并写入 `_state.licenseType`。

3. 执行限制动作
   - 当触发并发/授权限制时，会执行 `disableEditing(true)` 或 `disableLiveViewing(true)`，并主动 `asc_coAuthoringDisconnect()` 断开协作连接；同时给出提示文案。

4. 能力开关受 `canLicense` 控制
   - `canLicense` 由 license 结果计算，后续影响 `isEdit/canChat/canUseHistory/canComments` 等能力。

5. 错误码兜底
   - `Asc.c_oAscError.ID.UserCountExceed` 映射到用户数超限错误提示。

### 其他编辑器

- `spreadsheeteditor/main/app/controller/Main.js`
- `presentationeditor/main/app/controller/Main.js`
- `pdfeditor/main/app/controller/Main.js`
- `visioeditor/main/app/controller/Main.js`

这些文件都实现了与文档编辑器同构的 `onLicenseChanged/applyLicense/canLicense` 流程。
