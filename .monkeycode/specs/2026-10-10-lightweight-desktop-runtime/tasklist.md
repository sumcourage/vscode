# 需求实施计划

- [ ] 1. 抽出 desktop-host 分层与宿主无关类型
  - [ ] 1.1 在 `build/checker/layersChecker.ts`、`build/checker/layersTypeCheck.ts`、`.github/instructions/source-code-organization.instructions.md` 增加 `desktop-main` / `desktop-browser` / `desktop-utility` 分层规则，允许与现有 `electron-*` 并行（Requirement 3、12.4）
  - [ ] 1.2 新增 `src/vs/platform/window/common/desktopWindow.ts`，定义 `IDesktopWindowHandle`（id / kind / state），把 `IWindowState` 从 `src/vs/platform/window/electron-main/window.ts` 上移到 common（Requirement 6、design Data Models）
  - [ ] 1.3 新增 `src/vs/platform/utilityProcess/common/sidecarProcess.ts`，定义 `ISidecarProcessConfiguration`（type / name / entryPoint / args / env / parentLifecycleBound / payload）（Requirement 7、13）
  - [ ] 1.4 把 `IWindowsMainService` 的窗口句柄从 Electron `BrowserWindow` 改为 `IDesktopWindowHandle`，Electron Dual-Run 实现做 id 映射（Requirement 6）
  - [ ]* 1.5 为 `IDesktopWindowHandle` 与 `ISidecarProcessConfiguration` 序列化往返编写单元测试（Correctness: window state round-trip）

- [ ] 2. 抽出 vscode: IPC 与 preload 宿主无关桥
  - [ ] 2.1 把 `src/vs/base/parts/sandbox/electron-browser/preload.ts` 的 `validateIPC`（仅允许 `vscode:` 前缀）和 `ISandboxConfiguration` 解析抽到 `src/vs/base/parts/sandbox/common/hostBridge.ts`（Requirement 7.3、10.1、10.4）
  - [ ] 2.2 新增 `src/vs/base/parts/sandbox/desktop-browser/preload.ts`，在 OS webview 隔离上下文中拉取 `ISandboxConfiguration`、应用 `userEnv` 与 zoom，并暴露 allowlist IPC shim（Requirement 2.3、3.1、10）
  - [ ] 2.3 保留 Electron preload 在 Dual-Run 期间调用同一 `hostBridge` 校验逻辑（Requirement 12.4）
  - [ ]* 2.4 为 allowlist 拒绝非 `vscode:` 通道、以及 sandbox 配置缺失时的错误路径编写单元测试（Requirement 10.4）

- [ ] 3. 检查点 - 确保分层检查与现有单元测试通过
  - 运行 `npm run valid-layers-check` 与 `scripts/test.sh` 中 sandbox / window 相关选择器；如有疑问请询问用户

- [ ] 4. 落地 Tauri 2 特权宿主骨架
  - [ ] 4.1 新增 `src-tauri/` crate（`Cargo.toml`、`tauri.conf.json`、`src/lib.rs` / `src/main.rs`），配置 Windows WebView2 / macOS WKWebView / Linux WebKitGTK（Requirement 2.1、2.2、9.1）
  - [ ] 4.2 实现 Rust 启动：CLI argv、user-data 路径、单实例锁、portable 模式，对位 `src/mainImpl.ts` 与 `src/vs/code/electron-main/main.ts` 的现有行为（Requirement 2.2、12.1）
  - [ ] 4.3 新增 `src/vs/code/desktop-main/` TypeScript 组合根，通过宿主客户端调用 Rust 命令；业务规则留在 TypeScript（Requirement 3.2）
  - [ ] 4.4 实现工作台 webview 加载现有 `workbench.desktop.main.js`，不重写编辑器 UI（Requirement 2.4、3.1）
  - [ ]* 4.5 为 argv / user-data / portable 路径解析编写单元测试

- [ ] 5. 实现窗口管理与多窗口
  - [ ] 5.1 在 Rust 中实现 `WebviewWindow` 创建 / 关闭 / 聚焦，映射 `IDesktopWindowHandle`（Requirement 6.1）
  - [ ] 5.2 实现 auxiliary window 与 agents window 的创建与跟踪，对位 `src/vs/platform/auxiliaryWindow/electron-main/` 与 `agentsWindow.ts`（Requirement 6.2）
  - [ ] 5.3 实现最后窗口关闭时的 quit-or-keep-running，对齐 Compatibility Baseline 分平台生命周期（Requirement 6.3）
  - [ ] 5.4 持久化并恢复 window state、zoom、fullscreen、native title bar（Requirement 6.4）
  - [ ]* 5.5 为窗口状态往返与多窗口 id 分配编写单元测试（Correctness: window state round-trips）

- [ ] 6. 实现 ICommonNativeHostService 的 Tauri 后端
  - [ ] 6.1 新增 `src/vs/platform/native/desktop-main/nativeHostMainService.ts`，对 `ICommonNativeHostService` 每个 Compatibility Baseline 已用方法提供 Tauri command 实现（Requirement 2.3、8）
  - [ ] 6.2 实现菜单、上下文菜单、file/open/save/message 原生对话框（Requirement 8.1）
  - [ ] 6.3 实现 OS 剪贴板文本与文件（Requirement 8.2）
  - [ ] 6.4 实现 `code-oss` protocol URL 与文件关联，交给现有 URL / file-open 流程（Requirement 8.3）
  - [ ] 6.5 实现 OS keychain 安全存储，替代 Electron `safeStorage`（Requirement 8.4）
  - [ ] 6.6 实现全局快捷键、recent documents、jump list / dock badge 的分平台员工可见集成（Requirement 8.5）
  - [ ]* 6.7 为 native host 客户端假宿主单元测试覆盖每个已用方法（Correctness: every ICommonNativeHostService method has Tauri backend）

- [ ] 7. 实现 Node sidecar 进程监督器
  - [ ] 7.1 在 Rust 中实现 sidecar 启动 / 跟踪 / 重启，替代 `src/vs/platform/utilityProcess/electron-main/utilityProcess.ts`（Requirement 7.1、7.2、13.2）
  - [ ] 7.2 宿主与 sidecar 复用现有 Node IPC（`src/vs/base/parts/ipc/node/ipc.net.ts`）socket/pipe 握手（Requirement 7.3）
  - [ ] 7.3 sidecar 异常退出时复用现有 process-kind 重启或错误 UX（Requirement 7.4）
  - [ ] 7.4 Extension Host 保持独立进程，不与特权宿主合并（Requirement 10.3、13.2）
  - [ ]* 7.5 为 sidecar descriptor 序列化、重启策略、父进程绑定编写单元测试

- [ ] 8. 检查点 - 确保宿主、窗口、native host、sidecar 测试通过
  - 确保所有测试通过,如有疑问请询问用户

- [ ] 9. 接入 Extension Host / shared / PTY / agent 到 Bundled Node Runtime
  - [ ] 9.1 将 Extension Host 入口切到 Bundled Node Runtime，保留 `src/vs/workbench/api/node` API provider（Requirement 1.4、13.2、13.3）
  - [ ] 9.2 将 `sharedProcessMain.ts` 切到 desktop-utility + Bundled Node Runtime（Requirement 7.2）
  - [ ] 9.3 将 PTY host（`electronPtyHostStarter` / node-pty）切到 sidecar（Requirement 1.3、7.2）
  - [ ] 9.4 将 agent host（`electronAgentHostStarter`）切到 sidecar（Requirement 7.2）
  - [ ] 9.5 针对 Bundled Node ABI 重建并打包 `@vscode/spdlog`、`@vscode/sqlite3`、`@vscode/windows-*`、`@parcel/watcher`、`node-pty`（Requirement 13.1、13.3）
  - [ ]* 9.6 增加加载 native Node addon 的 fixture 扩展集成测试（Requirement 13.3）

- [ ] 10. 实现自定义协议、来源门与日志/崩溃
  - [ ] 10.1 用 Tauri custom protocol 提供 `vscode-file` / `vscode-webview` / `vscode-app` 等价物，复用 `protocolMainService.ts` 的 trusted-origin 规则（Requirement 10.2）
  - [ ] 10.2 非 allowlist 的 native 请求由桥拒绝，webview 走现有错误路径（Requirement 10.4）
  - [ ] 10.3 日志写到 Compatibility Baseline 的 user-data 日志目录（Requirement 11.1）
  - [ ] 10.4 特权宿主或 webview 崩溃时在 user-data crash 目录留下 dump（Requirement 11.2）
  - [ ] 10.5 Process Explorer / 诊断命令列出 host、workbench、Extension Host、shared process、PTY host（Requirement 11.3）
  - [ ]* 10.6 为协议 allowlist 与 untrusted origin 拒绝编写单元测试

- [ ] 11. 实现内置浏览器的额外 OS WebView
  - [ ] 11.1 把 Simple Browser / `browserView` 映射为 Tauri 宿主拥有的额外 OS webview 窗口或窗格（Requirement 14.1）
  - [ ] 11.2 对该 webview 应用 Compatibility Baseline 的 trusted-origin 与权限规则（Requirement 14.2）
  - [ ] 11.3 在 OS webview 提供等价调试协议时，将其暴露给现有自动化调用方；否则保持页面加载/导航/展示，CDP-only 自动化不可用（Requirement 14.3、14.4）
  - [ ]* 11.4 为 Simple Browser 加载、导航、展示编写集成测试

- [ ] 12. Dual-Run 开发启动路径
  - [ ] 12.1 新增 `scripts/code-tauri`（及 bat/sh），编译 client 后用 Tauri 加载同一 `out/`（Requirement 12.1、12.4）
  - [ ] 12.2 保留现有 `scripts/code.sh` Electron Dual-Run 启动（Requirement 12.4）
  - [ ] 12.3 确认 CLI（`src/vs/code/node/cli.ts`）、`code-server`、web workbench 入口不依赖 `src-tauri` 即可构建（Requirement 3.3）
  - [ ]* 12.4 为 CLI / server / web 入口做不依赖 Tauri crate 的构建冒烟

- [ ] 13. 检查点 - 确保 Dual-Run 启动与 sidecar 进程测试通过
  - 确保所有测试通过,如有疑问请询问用户

- [ ] 14. 打包 Bundled Node Runtime 与 Tauri 安装包
  - [ ] 14.1 实现 Node 运行时下载/暂存，ABI 对齐 Compatibility Baseline Electron 的 Node major，写入安装包（Requirement 13.1、13.4）
  - [ ] 14.2 新增 `gulp vscode-tauri-win32|darwin|linux`：暂存 workbench + Bundled Node + native addons，再跑 `tauri build`（Requirement 9.1、9.2、12.2）
  - [ ] 14.3 Windows 安装包检测 WebView2 Evergreen，缺失时跑官方 bootstrapper；Linux 包声明 `webkit2gtk` 依赖（Requirement 9.1、design Error Handling）
  - [ ] 14.4 产出代码签名 / notarization / Linux desktop-file 所需产物（Requirement 9.4）
  - [ ] 14.5 Tauri 员工包省略 Chromium 与 Electron main/renderer 二进制（Requirement 2.5）
  - [ ] 14.6 将 Tauri updater 接到现有 `IUpdateService`（Darwin / Win32）（Requirement 9.3）
  - [ ]* 14.7 为安装包清单断言“无 Electron/Chromium 二进制、含 Bundled Node”编写测试

- [ ] 15. 实现安装包体积与空闲内存预算记录
  - [ ] 15.1 新增 `IRuntimeBudgetRecord` 采集任务，记录 win32-x64 / darwin-arm64 / linux-x64 的 baseline 与 candidate 安装包字节、Idle RSS、中型工作区 RSS（Requirement 4.2、5.4）
  - [ ] 15.2 发布任务在 `candidateInstallerBytes > 0.2 * baselineInstallerBytes` 或 `candidateIdleRssBytes > 0.5 * baselineIdleRssBytes` 时失败（Requirement 4.3、5.1、5.2）
  - [ ] 15.3 中型工作区（约 5000 文件、一个 editor group、终端与资源管理器可见）RSS 必须低于 baseline（Requirement 5.3）
  - [ ]* 15.4 为预算比率门禁编写单元测试（固定 fixture 数字）

- [ ] 16. Dual-Run 自动化驱动与门禁后拆除 Electron
  - [ ] 16.1 为 `test/automation/src/electron.ts` 增加 Tauri driver，复用同一 workbench automation API；Dual-Run 期间保留 Electron driver（Requirement 12.4、1）
  - [ ] 16.2 能力、安装包、Idle Memory 三端门禁通过后，将默认员工打包路径切到 Tauri，移除 `@vscode/gulp-electron`、Electron 下载与 Chromium 产物（Requirement 12.3、12.5、2.5）
  - [ ] 16.3 更新默认桌面启动/打包命令文档路径为 Tauri（Requirement 12.3）
  - [ ]* 16.4 为“员工 Tauri 包不含 Electron/Chromium 二进制”编写回归断言
