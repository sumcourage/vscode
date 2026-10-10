# Requirements Document

## Introduction

This fork currently ships the Code - OSS desktop product on Electron. Electron bundles Chromium and Node.js into every installer, which sets a high floor for package size and resident memory.

The desktop product SHALL replace the Electron shell with Tauri 2 while preserving the existing employee-facing desktop capabilities, including the full extension ecosystem. The TypeScript workbench, editor, extension API surface, CLI, and remote/server stacks remain the product surface. Only the desktop process model, windowing, native OS integration, and packaging toolchain change.

Locked decisions:

- Desktop host: Tauri 2 with the OS webview.
- Extension compatibility: the full extension ecosystem, including extensions that load native Node addons. The Extension Host continues to run on a Node.js sidecar.
- Platforms: Windows, macOS, and Linux in the same release train.
- Size: each platform installer is at most 20 percent of the Compatibility Baseline installer.
- Memory: Idle Memory is at most 50 percent of the Compatibility Baseline Idle Memory.
- Migration: extract a desktop-host abstraction first; Electron and Tauri dual-run until capability and budget gates pass, then remove Electron packaging.
- Node sidecar: the Installer bundles one Node.js runtime aligned to the current Electron ABI.
- In-app browser: Simple Browser and related views use an extra OS webview window or pane. Chromium DevTools Protocol automation is enabled only when the OS webview exposes an equivalent protocol.

## Glossary

- **Desktop Product**: The local GUI application employees launch on Windows, macOS, and Linux, currently built from `src/vs/code/electron-*`, `src/vs/platform/*/electron-*`, and the Electron packaging pipeline.
- **Workbench**: The existing TypeScript UI and service layer under `src/vs/workbench`, `src/vs/editor`, `src/vs/platform`, and `src/vs/sessions`.
- **Electron Shell**: The current desktop host that owns the main process, renderer windows, utility processes, Chromium, and Node.js runtime.
- **Lightweight Desktop Runtime**: Tauri 2. The replacement host that provides windows, web content, IPC, native OS integration, and packaging. It uses the OS webview (WebView2 on Windows, WKWebView on macOS, WebKitGTK on Linux) and a Rust privileged host.
- **Node Sidecar**: A shipped Node.js runtime used for the Extension Host, shared process, PTY host, agent host, and other Node-backed desktop processes that the Compatibility Baseline currently runs in Electron utility or Node processes.
- **Employee-Facing Capability**: A capability employees already use in the current desktop product, including editing, file management, search, terminal, Git, debug, extensions, multi-window, settings, themes, protocol URLs, auto-update, and OS integration.
- **Extension Host**: The process that loads and runs VS Code extensions against the existing Extension API.
- **Native Capability**: An OS-level capability currently reached through Electron APIs, including windows, menus, dialogs, clipboard, protocol handlers, secure storage, process spawning, file dialogs, global shortcuts, power events, and auto-update.
- **Installer**: The platform-specific desktop package employees install (for example `.exe` / `.msi`, `.dmg` / `.zip`, `.deb` / `.rpm` / `.tar.gz`).
- **Idle Memory**: Resident set size of the Desktop Product after workbench first paint, with one empty window, no folder, and no extra extensions beyond built-in extensions.
- **Compatibility Baseline**: The current Electron-based Desktop Product on the same commit, used as the functional and packaging reference.
- **Dual-Run Period**: The interval after the desktop-host abstraction exists and before Electron packaging is removed, during which maintainers can launch both the Electron shell and the Tauri 2 host against the same workbench output.
- **Bundled Node Runtime**: The Node.js executable and matching ABI modules copied into the Installer and used exclusively as the Node Sidecar.
- **In-App Browser**: The Simple Browser and related `browserView` surfaces. In the Lightweight Desktop Runtime they render in an extra OS webview owned by the Tauri 2 host.

## Requirements

### Requirement 1: Preserve employee-facing desktop capabilities

**User Story:** AS an employee using the desktop editor, I want the existing product capabilities to keep working, so that the runtime change does not change daily work.

#### Acceptance Criteria

1. WHEN an employee launches the Desktop Product, the Desktop Product SHALL present the existing workbench, editor, activity bar, side bar, panel, status bar, and command palette.
2. WHEN an employee opens a folder, file, or workspace, the Desktop Product SHALL provide the Compatibility Baseline file explorer, editor groups, search, problems, output, and SCM views.
3. WHEN an employee uses terminal, Git, debug, tasks, settings, keybindings, or themes, the Desktop Product SHALL provide the Compatibility Baseline behavior for that capability.
4. WHEN an employee installs, enables, disables, or runs an extension that the Compatibility Baseline supports, including an extension that loads a native Node addon, the Desktop Product SHALL load that extension through the Node Sidecar Extension Host and expose the existing Extension API surface used by that extension.
5. IF a capability in the Compatibility Baseline depends on a Native Capability, the Tauri 2 privileged host SHALL provide an equivalent Native Capability so the employee-facing behavior remains available.

### Requirement 2: Replace the Electron shell

**User Story:** AS a product owner, I want the desktop host to stop shipping Electron as the runtime, so that installer size and memory are no longer bounded by a bundled Chromium.

#### Acceptance Criteria

1. WHEN the Desktop Product is packaged for employees, the Installer SHALL use Tauri 2 as the windowing and web-content host.
2. WHEN the Desktop Product starts, the Tauri 2 privileged host SHALL own process lifetime, window creation, web-content loading, and native OS integration.
3. WHEN workbench TypeScript code requests a Native Capability, the Desktop Product SHALL route that request through a host abstraction that the Tauri 2 privileged host implements.
4. WHILE the Desktop Product runs as a packaged employee build, the Desktop Product SHALL render workbench UI in the OS webview provided by Tauri 2.
5. WHEN the Desktop Product is packaged, the Installer SHALL omit Chromium and the Electron main/renderer binaries.

### Requirement 3: Keep the existing TypeScript product surface

**User Story:** AS a maintainer of this fork, I want the workbench and editor TypeScript layers to stay the product UI, so that employee workflows and future VS Code merges remain tractable.

#### Acceptance Criteria

1. WHEN the Lightweight Desktop Runtime hosts the UI, the Desktop Product SHALL load the existing workbench and editor TypeScript surface rather than a rewritten editor UI.
2. WHEN platform services currently split across `common`, `browser`, `node`, and desktop-host layers are used, the Desktop Product SHALL keep `common`, `browser`, and `node` service contracts intact and swap only the desktop-host implementations.
3. WHEN CLI, `code-server`, and web workbench entry points are built, those entry points SHALL continue to build and run independently of the Lightweight Desktop Runtime.

### Requirement 4: Smaller installer

**User Story:** AS an employee or IT administrator, I want a smaller desktop installer, so that download, deploy, and disk use are cheaper.

#### Acceptance Criteria

1. WHEN the Desktop Product is packaged for Windows x64, macOS arm64, and Linux x64, each platform Installer SHALL have a smaller byte size than the Compatibility Baseline installer for that platform.
2. WHEN a release candidate is produced, the build SHALL record Compatibility Baseline installer size and Lightweight Desktop Runtime installer size per platform.
3. WHEN comparing those recorded sizes, the Lightweight Desktop Runtime installer SHALL be at most 20 percent of the Compatibility Baseline installer size on each measured platform.

### Requirement 5: Lower memory use

**User Story:** AS an employee, I want the desktop editor to use less RAM at idle and under normal editing, so that the machine stays responsive with other tools open.

#### Acceptance Criteria

1. WHEN the Desktop Product reaches Idle Memory on a measured platform, Idle Memory SHALL be lower than the Compatibility Baseline Idle Memory on that platform.
2. WHEN Idle Memory is compared with the Compatibility Baseline, Idle Memory SHALL be at most 50 percent of the Compatibility Baseline Idle Memory on each measured platform.
3. WHEN an employee opens a medium workspace of about 5,000 files with one editor group, integrated terminal, and file explorer visible, the Desktop Product resident set size SHALL stay below the Compatibility Baseline resident set size for the same scenario.
4. WHEN memory measurements are taken, the build or performance harness SHALL record both Compatibility Baseline and Lightweight Desktop Runtime figures for Idle Memory and the medium-workspace scenario.

### Requirement 6: Native windowing and multi-window

**User Story:** AS an employee, I want multiple editor windows and auxiliary windows to behave as they do today, so that multi-root and auxiliary workflows stay intact.

#### Acceptance Criteria

1. WHEN an employee opens a new window, the Lightweight Desktop Runtime SHALL create a distinct native window that loads the workbench.
2. WHEN an employee opens an auxiliary window that the Compatibility Baseline supports, the Desktop Product SHALL create and track that auxiliary window.
3. WHEN the last window closes according to the platform window-lifecycle rules used by the Compatibility Baseline, the Desktop Product SHALL apply the same quit-or-keep-running behavior.
4. WHEN window state, zoom, fullscreen, or native title bar configuration changes, the Desktop Product SHALL persist and restore that window state on the next launch as the Compatibility Baseline does.

### Requirement 7: IPC and process model

**User Story:** AS a maintainer, I want a documented process model that still isolates UI, extension execution, shared services, PTY, and agent host, so that capability and security boundaries remain.

#### Acceptance Criteria

1. WHEN the Desktop Product starts, Tauri 2 SHALL provide a Rust privileged host process and at least one OS webview process for the workbench.
2. WHEN the Extension Host, shared process, PTY host, or agent host is required by an employee-facing capability, the Desktop Product SHALL start that process as a Node Sidecar and connect it to the workbench through IPC.
3. WHEN workbench code currently uses desktop IPC channels, the Desktop Product SHALL preserve those channel contracts at the TypeScript service boundary.
4. IF a child process exits unexpectedly, the Desktop Product SHALL surface the existing restart or error behavior for that process kind.

### Requirement 8: Native OS integration

**User Story:** AS an employee, I want menus, dialogs, clipboard, protocol URLs, and file associations to keep working, so that the editor still behaves as a native desktop application.

#### Acceptance Criteria

1. WHEN an employee uses application menus, context menus, or file/open/save/message dialogs, the Desktop Product SHALL present native OS dialogs and menus equivalent to the Compatibility Baseline.
2. WHEN an employee copies or pastes text or files, the Desktop Product SHALL use the OS clipboard.
3. WHEN the OS opens a `code-oss` protocol URL or a registered file type, the Desktop Product SHALL hand that target to the existing URL and file-open flow.
4. WHEN the Desktop Product needs secure storage for secrets that the Compatibility Baseline stores with OS-backed encryption, the Lightweight Desktop Runtime SHALL store those secrets with OS-backed encryption.
5. WHEN the Compatibility Baseline registers global shortcuts, recent documents, or jump-list entries on a platform, the Desktop Product SHALL provide the same employee-visible OS integration on that platform.

### Requirement 9: Packaging, update, and supported platforms

**User Story:** AS an IT administrator, I want installers and updates for the same platforms employees already use, so that rollout stays operationally familiar.

#### Acceptance Criteria

1. WHEN a release is built, the build SHALL produce Desktop Product packages for Windows, macOS, and Linux in the same release train.
2. WHEN an employee installs a package, the installed application SHALL launch the Tauri 2 Desktop Product.
3. WHEN the Compatibility Baseline auto-update flow is enabled for a platform, the Desktop Product SHALL check for updates and apply an update without requiring the employee to manually replace files.
4. WHEN code signing, notarization, or Linux desktop-file metadata is required by the target platform store or OS, the packaging pipeline SHALL emit the artifacts those platform rules require.

### Requirement 10: Security and sandboxing

**User Story:** AS a security reviewer, I want the replacement host to keep web content isolated from privileged native APIs, so that renderer compromise stays contained.

#### Acceptance Criteria

1. WHEN workbench web content runs, the Lightweight Desktop Runtime SHALL expose native APIs only through an explicit, allowlisted host bridge.
2. WHEN a custom protocol or local file is loaded into web content, the Desktop Product SHALL apply the existing trusted-origin rules used by the Compatibility Baseline protocol service.
3. WHEN an extension runs, the Extension Host SHALL remain a process separate from the privileged host process.
4. IF web content requests a Native Capability outside the allowlist, the Lightweight Desktop Runtime SHALL reject that request and leave the Native Capability unavailable to that web content.

### Requirement 11: Observability and diagnostics

**User Story:** AS a maintainer, I want crash reports, logs, and process diagnostics to remain available, so that employee issues can still be triaged.

#### Acceptance Criteria

1. WHEN the Desktop Product runs, the Desktop Product SHALL write logs to the existing user-data log location used by the Compatibility Baseline.
2. WHEN a privileged host or web-content process crashes, the Desktop Product SHALL capture a crash report or equivalent dump that maintainers can retrieve from the user-data location.
3. WHEN an employee opens Process Explorer or the existing diagnostics commands, the Desktop Product SHALL list the Lightweight Desktop Runtime processes that correspond to host, workbench, Extension Host, shared process, and PTY host.

### Requirement 12: Build and developer inner loop

**User Story:** AS a fork maintainer, I want a documented way to compile, launch, and package the Lightweight Desktop Runtime product, so that daily development remains possible.

#### Acceptance Criteria

1. WHEN a maintainer runs the documented desktop development launch command, the Desktop Product SHALL start under the Lightweight Desktop Runtime against the compiled workbench output.
2. WHEN a maintainer runs the documented package command, the build SHALL produce platform installers that satisfy Requirement 4.
3. WHEN Electron-specific gulp tasks and `@vscode/gulp-electron` are no longer the desktop packaging path, the documented build commands SHALL describe the Lightweight Desktop Runtime packaging path as the default desktop path.
4. WHILE the Dual-Run Period is active, a maintainer SHALL be able to launch the same compiled workbench through either the Electron shell or the Tauri 2 host.
5. WHEN capability, installer-size, and Idle Memory gates pass on Windows, macOS, and Linux, the default employee packaging path SHALL be Tauri 2 and the Electron packaging path SHALL be removed.

### Requirement 13: Bundled Node sidecar

**User Story:** AS an employee using third-party extensions, I want native Node addons to load without depending on a system Node install, so that the full extension ecosystem keeps working on managed machines.

#### Acceptance Criteria

1. WHEN the Desktop Product is packaged, the Installer SHALL contain one Bundled Node Runtime whose ABI matches the Node major used by the Compatibility Baseline Electron build.
2. WHEN the Extension Host, shared process, PTY host, or agent host starts, the Desktop Product SHALL execute that process with the Bundled Node Runtime.
3. WHEN an extension loads a native Node addon built for that ABI, the Extension Host SHALL load the addon.
4. WHEN installer size is measured against Requirement 4, the Bundled Node Runtime bytes SHALL be included in the candidate installer size.

### Requirement 14: In-app browser on OS webview

**User Story:** AS an employee, I want Simple Browser to remain available, so that in-editor web preview keeps working after Chromium is removed from the installer.

#### Acceptance Criteria

1. WHEN an employee opens Simple Browser or a related in-app browser view, the Desktop Product SHALL present that content in an extra OS webview window or pane owned by the Tauri 2 host.
2. WHEN that extra OS webview loads a page, the Desktop Product SHALL apply the same trusted-origin and permission rules used by the Compatibility Baseline browser view.
3. WHEN the OS webview exposes a debugging protocol equivalent to the Compatibility Baseline CDP surface, the Desktop Product SHALL offer that protocol to the existing automation callers.
4. IF the OS webview omits an equivalent debugging protocol, the Desktop Product SHALL keep Simple Browser page load, navigation, and display available and leave CDP-only automation unavailable.
