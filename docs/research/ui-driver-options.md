# UI 驱动三选一：`winapp ui`、FlaUI、`UIAutomationClient`

对应 issue [#2](https://github.com/trytoreachpeak0/8005-test-lab/issues/2)，2026-09-04 调研。
只读了一手来源：三者的仓库源码与官方文档、Microsoft Learn、以及本工作区里已经在用它们的代码。
没有在任何机器上安装或运行过这三样东西，所以凡是「实测才能知道」的点都放在最后一节。

## 问题

lab 要让 agent 用一套 CLI 驱动 WPF 客户端（`SQCD.Agv.Wpf`）：`invoke` / `set-value` /
`inspect` / `wait-for` / `screenshot`。客户端跑在会动的真车上位机和 Windows 11 VM 上，调用方
经 sshd 进来落在 session 0，窗口在 session 1；有时窗口没有焦点，有时桌面锁着。要回答的是：
三种驱动方式各自能做到什么，lab 该选哪一种，以及如果选 FlaUI，`lab-ui` 这个 .NET 8 JSON
包装器必须自己补哪些东西。

三个候选：

| 候选 | 是什么 | 本工作区里的现状 |
| --- | --- | --- |
| `winapp ui` | Microsoft `winappCli` 的一组子命令，一个原生 exe | 没用过 |
| FlaUI | .NET 库（`FlaUI.Core` + `FlaUI.UIA3`），封装 UIA3 COM API | `8005-mes-ingest/MesIngest.Watch.UiTests` 用 5.0.0 |
| `UIAutomationClient` | .NET 自带的托管 UIA（UIA2），从 pwsh `Add-Type` 直接用 | `8005-agv-control-server/scripts/l2/L2.psm1` 第 405-530 行 |

## 三者对比表

| 维度 | `winapp ui` | FlaUI 5.0.0 | `UIAutomationClient`（pwsh） |
| --- | --- | --- | --- |
| 分发形态 | NativeAOT 单文件 exe，自包含，不依赖 .NET 运行时 [W1] | 库。要么写宿主程序（`lab-ui`）再 `dotnet publish` 成单文件，要么把三个 dll 拷过去从 pwsh 加载 | 零安装：pwsh 7.6.5 的 `$PSHOME` 自带 `UIAutomationClient.dll` [U1]，每台目标机都有 pwsh 7.6.5 |
| 目标机预装 | 不需要，scp 一个 zip 解开即可；x64 zip 约 46.7 MB [W2] | 不需要（发布成自包含单文件后） | 不需要 |
| `invoke` 无焦点 | 可（UIA InvokePattern，不注入输入）[W3] | 可（`AsButton().Invoke()`）[F1] | 可，L2 已在用 [U2] |
| `set-value` 无焦点 | 可（ValuePattern → RangeValue → LegacyIAccessible 链）[W4] | 可（`Patterns.Value.Pattern.SetValue`） | 可，L2 已在用 [U2] |
| `inspect` | 有：树、`--interactive`、`--json`、语义 slug [W5] | 无现成命令，`TreeWalker` 自己走；mes-ingest 已写过一份 `DumpUiaTree` [F2] | 无，`FindAll` + `Current.*` 自己拼 |
| `wait-for` | 有：元素出现/消失/属性值，超时退出码 1，`--json` 信封 [W6] | `Retry.WhileFalse` / `WhileNull`，默认超时 1 s 间隔 100 ms [F3] | 无，L2 用 `Start-Sleep` 循环手写 [U3] |
| `screenshot` 无焦点/被遮挡 | 可：默认 Windows.Graphics.Capture，被遮挡也能截；退化到 `PrintWindow` [W7] | **不可靠**：`Capture.Element` 走桌面 DC `BitBlt`，截的是屏幕区域，被遮挡就截到别的窗口 [F4]；mes-ingest 因此自己 P/Invoke `PrintWindow` [F5] | 无，要 `Add-Type` 一段 C# P/Invoke `PrintWindow` |
| 锁屏时的模式动词 | 文档明说 `inspect/search/get-property/get-value/wait-for/set-value/invoke/screenshot` 是 "headless/locked-session friendly" [W3] | 库本身不判断桌面状态；UIA 模式调用理论上同 winapp，未实测 | 同左，未实测 |
| 锁屏时的注入动词 | `click/hover/drag/touch/pen/scroll --wheel/send-keys --via send-input` 快速失败 `no_interactive_desktop` [W3] | `Mouse.*` / `Keyboard.*` / `element.Click()` 走 `SendInput` [F6]，只进前台窗口 [M1]，锁屏时输入桌面是 Winlogon [M2]，没有失败检测 | 没有注入 API |
| session 0 → session 1 | 是 exe，可由计划任务在交互会话拉起；`--json` 写 stdout，计划任务不带回 stdout，要重定向到文件 [M3] | 同左，宿主程序要自己做 | 同左，pwsh 脚本 `ConvertTo-Json > file` |
| `AutomationId` 定位 | 一等公民：唯一时直接当选择器；无则语义 slug；还有 Name 子串匹配 [W8] | `ConditionFactory.ByAutomationId` [F7] | `PropertyCondition(AutomationIdProperty, …)` [U2] |
| 客户端只有 `x:Name` 的影响 | 无影响：WPF 没设 `AutomationId` 时用 `x:Uid`，再没有就用 `Name` [X1] | 同左 | 同左，L2 的 `ScanTextBox` 就是这样命中的 |
| WPF 支持边界 | 官方表：WPF `inspect/search/invoke/set-value/screenshot` 全勾；`RichTextBox` 不能 `set-value` [W9] | UIA3 "works great for WPF" [F8] | 托管 UIA2；FlaUI 作者说 UIA2 "does not work well with WPF" [F8]，微软文档也把它定位为 .NET Framework 时代 API [U4]；但 L2 用到的 Value/Invoke/IsEnabled 实测能用 |
| 目标框架 | CLI `net10.0-windows10.0.19041.0`；库包只出 `net10.0-windows` [W10] | `net48;net6.0-windows;net8.0-windows` [F9]，与 ADR 0056 的 `net8.0` 基线吻合 | 跟随 pwsh（.NET 10 运行时） |
| 目标机最低 OS | TFM 19041；WGC 需 Windows 10 1803+ [M4]。真车上位机 Windows 10 22H2 (19045) 满足，VM 是 Windows 11 [S1] | 无额外要求 | 无额外要求 |
| 许可证 | MIT [W11] | MIT [F10] | .NET 的一部分，MIT |
| 成熟度 | **Public Preview**，最新 v0.6.0（2026-08-12）[W12]；遥测默认开 [W13] | 5.0.0 发布于 2025-02-25 [F11]，本工作区已经在生产门禁里跑 | 稳定但停止演进 [U4] |

引用编号在下一节展开。

## 逐项证据

### `winapp ui`

- **[W1] 分发形态。**CLI 工程 `src/winapp-CLI/WinApp.Cli/WinApp.Cli.csproj` 第 5 行
  `<TargetFramework>net10.0-windows10.0.19041.0</TargetFramework>`，第 22-24 行
  `<PublishAot>true</PublishAot>`、`<SelfContained>true</SelfContained>`、
  `<PublishSingleFile>true</PublishSingleFile>`
  （https://github.com/microsoft/winappCli/blob/main/src/winapp-CLI/WinApp.Cli/WinApp.Cli.csproj ）。
  README 第 179 行把 `cli-binaries.zip` 描述为 "Native CLI executables (win-x64, win-arm64)"
  （https://github.com/microsoft/winappCli/blob/main/README.md ）。
- **[W2] 安装渠道与体积。**README 第 143-168 行：`winget install Microsoft.winappcli`、
  `npm install @microsoft/winappcli`、或从 GitHub Releases 手动下载。
  `GET /repos/microsoft/winappCli/releases/latest`：`v0.6.0`，`winappcli-x64.zip` 46,672,666 字节，
  另有 `winappcli_x64.msix` 与 `Microsoft.Windows.SDK.BuildTools.WinApp.nupkg`。
- **[W3] 动词与桌面状态。**`docs/ui-automation.md` 第 16 行（`main` 与 `v0.6.0` 标签两份文档
  同一段）："`click`, `hover`, `drag`, `touch`, `pen`, `scroll --wheel`, and `send-keys --via send-input`
  synthesize OS-level input, so they need an **unlocked, interactive desktop** with the target window in
  the foreground. On a **locked workstation or secure desktop** (LogonUI/UAC) they can't inject and fail
  fast with **`no_interactive_desktop`** … Everything else — `inspect`, `search`, `get-property`,
  `get-value`, `wait-for`, `set-value`, `invoke`, `scroll --direction/--to`, `screenshot` — drives the
  app through UIA patterns and is **headless/locked-session friendly**."
  （https://github.com/microsoft/winappCli/blob/main/docs/ui-automation.md ）
- **[W4] `set-value`。**同文档第 451-455 行："Set a value on an editable element **programmatically**
  (no keystrokes, no app foreground)"，回退链 ValuePattern → RangeValuePattern → LegacyIAccessible。
- **[W5] `inspect`。**第 157-194 行：默认深度 3，`--interactive` 只列可 invoke 的元素，状态标记
  `[disabled]`、`[offscreen]`、`value="…"`。第 723-727 行：`inspect --json` 返回
  `{ windows: [{ hwnd, title, elements: [...] }] }`。
- **[W6] `wait-for`。**第 488-496 行：`--timeout`、`--value`、`--property`、`--gone`、`--contains`。
  第 711-714 行："when no element matches (`search`) or the wait times out (`wait-for`), the command
  writes a fully parseable result envelope to **stdout** … and returns **exit code 1**. Stderr is empty
  in `--json` mode".
- **[W7] `screenshot`。**第 240 行："The default capture path uses **Windows.Graphics.Capture (WGC)**,
  reading the actual DWM-composited surface — … working even while the window is occluded by other
  windows. If WGC is unavailable (older Windows builds) the CLI falls back to **PrintWindow**."
  第 242 行：`--capture-screen` 和 `--focus` 会把窗口拉到前台，默认路径不会。
- **[W8] 选择器。**第 73-84 行：三种选择器——唯一的 `AutomationId` 直接用；否则 `prefix-name-hash`
  语义 slug，hash 取自 `RuntimeId`，元素换了就报 "Element may have changed. Re-run inspect."；
  再否则按 Name/AutomationId 的大小写不敏感子串匹配。
- **[W9] WPF 边界。**第 546-555 行 Framework Support 表：WPF 五列全勾，脚注 "**WinUI 3 `RichEditBox`
  and WPF `RichTextBox` are exceptions** — they expose only the read-only Text pattern"。
  第 439 行：`send-keys --via post-message` 对 WPF 有效（"WPF windows are single-HWND and route keys
  to the internally focused element, so post-message works there"），且不要求前台、绕过 UIPI（第 436 行）。
- **[W10] 库包的目标框架。**第 578-580 行："The automation package targets both `net10.0-windows` and
  `net10.0-windows10.0.19041.0`"；`src/winapp-CLI/WinApp.UIAutomation/WinApp.UIAutomation.csproj`
  第 7 行 `<TargetFrameworks>net10.0-windows;net10.0-windows10.0.19041.0</TargetFrameworks>`。
  这意味着**只能当 exe 用，不能作为库引进 net8.0 工程**（ADR 0056 锁定 SDK 8.0.424 / `net8.0`，
  见 `repos/8005-agv-program/docs/adr/cross/0056-dotnet-toolchain-baseline.md` 第 3 行）。
- **[W11] 许可证。**`LICENSE`：MIT License, Copyright (c) Microsoft Corporation and Contributors。
- **[W12] 成熟度。**README 第 4 行："**Status: Public Preview** — The Windows App Development CLI
  (winapp CLI) is experimental and in active development."第 7 行提醒 `main` 的文档可能与已发布
  版本不同——已核对 `v0.6.0` 标签下的 `docs/ui-automation.md`（718 行）含同一段 [W3] 文字，
  `ui` 子命令在正式发布里。
- **[W13] 遥测。**`docs/telemetry.md` 第 32 行："The winapp CLI telemetry feature is enabled by default.
  To opt out … set the `WINAPP_CLI_TELEMETRY_OPTOUT` environment variable to `1`."
- 另一条与 pwsh 相关的提醒，第 652 行："PowerShell's `&&` operator can freeze when a native CLI
  writes to stderr or uses ANSI escape sequences. Use `;` instead"。

### FlaUI 5.0.0

- **[F1] 模式动词。**`MesIngest.Watch.UiTests/WatchWorkspaceProductionJourneyTests.cs` 第 1003 行
  `FindRequiredById(window, "ApplyHostButton").AsButton().Invoke();`，第 3089 行
  `range.SetValue(range.Minimum.ValueOrDefault)`。
- **[F2] 树导出。**`MesIngest.Watch.UiTests/WatchWindowJourneySupport.cs` 第 22-56 行
  `DumpUiaTree`：用 `automation.TreeWalkerFactory.GetControlViewWalker()` 递归，输出
  ControlType / AutomationId / Name / IsEnabled，上限 5000 个元素。lab 需要 `inspect` 时这段可以直接搬。
- **[F3] 等待。**`src/FlaUI.Core/Tools/Retry.cs`（v5.0.0）第 16、21 行
  `DefaultTimeout = 1000 ms`、`DefaultInterval = 100 ms`；第 133、143、188 行
  `WhileFalse` / `WhileNull` / `WhileException`
  （https://github.com/FlaUI/FlaUI/blob/v5.0.0/src/FlaUI.Core/Tools/Retry.cs ）。
- **[F4] 截图走屏幕 DC。**`src/FlaUI.Core/Capturing/Capture.cs`（v5.0.0）第 82-84 行
  `Element(...)` 直接 `Rectangle(element.BoundingRectangle, …)`；第 106-117 行用
  `Gdi32.BitBlt(…, CopyPixelOperation.SourceCopy | CopyPixelOperation.CaptureBlt)`；
  第 148-152 行 `CaptureDesktopToBitmap` 的源是 `User32.GetWindowDC(User32.GetDesktopWindow())`
  （https://github.com/FlaUI/FlaUI/blob/v5.0.0/src/FlaUI.Core/Capturing/Capture.cs ）。
  截的是屏幕矩形，与目标窗口是否被遮挡无关。
- **[F5] mes-ingest 的替代方案。**`MesIngest.Watch.UiTests/WatchWindowNative.cs` 第 165-172 行：
  "PrintWindow renders the client independently of desktop occlusion. A screen rectangle capture can
  silently include another foreground window and poison an otherwise valid pixel baseline."
  这是本工作区已经踩过并绕开的坑。
- **[F6] 输入注入。**`src/FlaUI.Core/Input/Mouse.cs`（v5.0.0）第 214、224 行 `SendInput(...)`；
  `Keyboard.cs` 第 36-37 行 `SendInput(character, …)`；
  `src/FlaUI.Core/AutomationElements/AutomationElement.cs` 第 181-183 行
  `Click(bool moveMouse)` → `PerformMouseAction(moveMouse, Mouse.LeftClick)`。
  `Focus()`（第 231 行）走 UIA `SetFocus`，不是注入。
- **[F7] `AutomationId` 定位。**`WatchWorkspaceProductionJourneyTests.cs` 第 1131 行
  `inspector.ConditionFactory.ByAutomationId(...)`。
- **[F8] UIA2 与 UIA3 的差别。**FlaUI README 第 27-30 行："UIA2 is managed only, which would be good
  for C# but it does not support newer features (like touch) and it also does not work well with WPF …
  UIA3 is the newest of them all and works great for WPF / Windows Store Apps"
  （https://github.com/FlaUI/FlaUI/blob/master/README.md ）。
- **[F9] 目标框架。**`src/FlaUI.UIA3/FlaUI.UIA3.csproj`（v5.0.0）第 4 行
  `<TargetFrameworks>net48;net6.0-windows;net8.0-windows</TargetFrameworks>`，第 60-65 行依赖
  `Interop.UIAutomationClient` 10.19041.0。`8005-mes-ingest/Directory.Packages.props` 第 27 行
  `<PackageVersion Include="FlaUI.UIA3" Version="5.0.0" />`。
- **[F10] 许可证。**`LICENSE.txt`：The MIT License (MIT), Copyright (c) 2016-present。
- **[F11] 版本。**`GET /repos/FlaUI/FlaUI/releases`：`v5.0.0` 2025-02-25，上一版 `v4.0.0` 2022-09-25。
- **主窗口识别的坑。**`src/FlaUI.Core/Application.cs`（v5.0.0）第 303-316 行 `GetMainWindow`
  依赖 `Process.MainWindowHandle`。L2.psm1 第 440-446 行注释记录了为什么这在 onboard 上不够：
  启动失败时 `App.xaml.cs` 先弹一个模态 `MessageBox`，进程唯一的窗口就是那个对话框。
  L2 的做法是「按需要的控件识别主窗口」，`lab-ui` 也得这么做，而不是用 `GetMainWindow`。
- **桌面会话门禁是宿主的事。**`8005-mes-ingest/Invoke-WatchUiTests.ps1` 第 105-118 行用
  `OpenInputDesktop` 判断有没有交互桌面，没有就退出码 2；
  `docs/agents/golden-renderer.md` 第 41-45 行："`ssh vm01` reaches the guest, but lands in session 0.
  That session has no interactive window station: WPF fails there with dispatcher thread-affinity
  errors that name nothing resembling the real cause"。FlaUI 本身没有这层判断。

### `UIAutomationClient`（从 pwsh）

- **[U1] 零安装。**控制端 `C:\Program Files\PowerShell\7\` 目录下有 `UIAutomationClient.dll`、
  `UIAutomationTypes.dll`、`UIAutomationClientSideProviders.dll`、`PresentationCore.dll`，
  `pwsh -NoProfile` 报 7.6.5 / `.NET 10.0.11`（2026-09-04 本机列目录）。工作区根 `CLAUDE.md`
  「Scripting baseline」一节：控制端、三台上位机、factory-server 都是 pwsh 7.6.5。
- **[U2] 已验证的动词。**`repos/8005-agv-control-server/scripts/l2/L2.psm1` 第 429 行
  `Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes`；第 405-407 行注释：
  "ValuePattern.SetValue on ScanTextBox and InvokePattern on the 「手动提交」 button both work on an
  unfocused window, so the run does not fight the operator's keyboard and does not break when
  something else takes focus."第 501-508 行 `SetSublot`、第 516-520 行 `Submit`、第 489-493 行
  `CanSubmit` 读 `Current.IsEnabled`、第 478-487 行 `Element` 用 `AutomationIdProperty` /
  `NameProperty` 的 `PropertyCondition`。
- **[U3] 没有 `wait-for`。**第 447-475 行 `Attach` 是手写的 `while ($true)` + `Start-Sleep 250 ms`
  轮询；所有等待逻辑都在场景脚本里，没有可复用的命令。
- **[U4] 定位为旧 API。**Microsoft Learn「UI Automation Overview」页首："This documentation is intended
  for .NET Framework developers who want to use the managed UI Automation classes defined in the
  `System.Windows.Automation` namespace. For the latest information about UI Automation, see Windows
  Automation API: UI Automation."
  （https://learn.microsoft.com/en-us/dotnet/framework/ui-automation/ui-automation-overview ）
  同页另一条限制："UI Automation does not enable communication between processes started by different
  users through the **Run as** command."
- **没有截图、没有树导出、没有 JSON 契约。**L2.psm1 第 411-414 行的设计是「Driving only, never
  asserting」，UI 只读一个 `IsEnabled`，一切结论来自数据库和模拟器快照。这对 L2 够用，对 lab 的
  `inspect` / `screenshot` 动词不够。

### 三者共用的事实：session 1、锁屏、提权

- **[M1] `SendInput` 进前台。**Microsoft Learn `SendInput`："This function is subject to UIPI. Applications
  are permitted to inject input only into applications that are at an equal or lesser integrity level."
  返回值一节："This function fails when it is blocked by UIPI. Note that neither GetLastError nor the
  return value will indicate the failure was caused by UIPI blocking."
  （https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput ）
- **[M2] 锁屏时输入桌面是 Winlogon。**Microsoft Learn「Desktops」："only one of these desktops at a time
  is active. This active desktop, also known as the *input desktop*, is the one that is currently visible
  to the user and that receives user input." "During the user's session, the system switches to the
  Winlogon desktop when the user presses the CTRL+ALT+DEL key sequence, or when the User Account Control
  (UAC) dialog box is open." "The Winlogon desktop's security descriptor allows access to a very
  restricted set of accounts … Applications generally … cannot access the Winlogon desktop or switch to a
  different desktop while the Winlogon desktop is active."同页："Window messages can be sent only
  between processes that are on the same desktop."
  （https://learn.microsoft.com/en-us/windows/win32/winstation/desktops ）
  后一句是「session 0 的 sshd 进程不能直接驱动 session 1 窗口」的根据。
- **[M3] 计划任务拉起交互会话。**`remote-ops/onboard-hmi/scripts/10-start-onboard-stack.ps1`
  第 111-140 行 `Start-InInteractiveSession`：`query session` 找 Active 的 console 用户，
  `New-ScheduledTaskPrincipal -UserId $user -LogonType Interactive -RunLevel Limited`，
  `Register-ScheduledTask` → `Start-ScheduledTask` → 3 秒后 `Unregister-ScheduledTask`。
  Task Scheduler 文档 `TASK_LOGON_INTERACTIVE_TOKEN`："User must already be logged on. The task will
  be run only in an existing interactive session."
  （https://learn.microsoft.com/en-us/windows/win32/taskschd/principal-logontype ）
  `Get-ScheduledTaskInfo` 只返回 `LastRunTime` / `LastTaskResult` 等运行时信息
  （https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/get-scheduledtaskinfo ），
  **任务进程的 stdout 不会带回来**。三种方案要把 JSON 带回 session 0 的调用方，都得让任务把输出
  写到文件，调用方轮询任务状态再读文件。10-start 的「3 秒后注销」对 UI 动词不适用——动词可能
  跑几十秒，任务要活到进程退出。
- **[M4] WGC 最低版本。**`Windows.Graphics.Capture.GraphicsCaptureSession` 要求
  "Windows 10, version 1803 (introduced in 10.0.17134.0)"
  （https://learn.microsoft.com/en-us/uwp/api/windows.graphics.capture.graphicscapturesession ）。
- **[S1] 目标机 OS。**`remote-ops/onboard-hmi/README.md` 第 59 行：上位机
  "Windows 10 专业版 22H2 (10.0.19045)"；工作区根 `CLAUDE.md`「Lab VMs」：`win11-01` 是
  Windows 11 Pro 26200。两者都 ≥ 19041。
- **提权边界。**Microsoft Learn「Security Considerations for Assistive Technologies」："An application that
  doesn't have UIAccess in the manifest starts with medium IL and cannot access elevated ("medium+" IL)
  process UI."
  （https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-securityoverview ）
  客户端由 10-start 以 `-RunLevel Limited` 拉起，驱动程序只要同样不提权就在同一 IL。
- **[X1] WPF 的 `AutomationId` 回退。**`dotnet/wpf`
  `src/Microsoft.DotNet.Wpf/src/PresentationFramework/System/Windows/Automation/Peers/FrameworkElementAutomationPeer.cs`
  第 15-30 行 `GetAutomationIdCore`：先取 `AutomationProperties.AutomationId`，空则取 `x:Uid`，
  再空则取 `FrameworkElement.Name`
  （https://github.com/dotnet/wpf/blob/main/src/Microsoft.DotNet.Wpf/src/PresentationFramework/System/Windows/Automation/Peers/FrameworkElementAutomationPeer.cs ）。
  `repos/8005-agv-onboard-hmi/src/SQCD.Agv.Wpf/MainWindow.xaml` 只有第 84 行
  `x:Name="ScanTextBox"` 和第 241 行 `x:Name="LogListBox"`，两个按钮（第 100 行 `Content="手动提交"`、
  第 146 行 `Content="申请恢复"`）没有名字，只能按 Name 定位——三种方案在这一点上完全一样，
  改善的办法也一样：在 `w2g/*` 分支给按钮加 `AutomationProperties.AutomationId`。

## 结论与推荐

**推荐 `winapp ui` 作为 lab 的 UI 动词执行器，不写 `lab-ui`。**理由按权重：

1. issue 列的五个动词它都有，而且 `inspect` / `wait-for` / `screenshot` 三个恰好是另外两条路
   没有、要自己写的 [W5][W6][W7]。`wait-for` 的退出码与 `--json` 信封 [W6] 就是 lab 需要的
   「等判据」契约。
2. 桌面状态的边界是文档明确承诺的 [W3]：模式动词锁屏可用，注入动词快速失败并给出可判别的错误码。
   FlaUI 和裸 UIA 在锁屏下的行为没有文档承诺，只能实测。
3. 一个自包含原生 exe [W1]，scp 到目标机即可用，与「不在真车上装东西」的约束相容。
4. 截图默认 WGC，被遮挡可截、不抢前台 [W7]。FlaUI 的截图是屏幕 `BitBlt` [F4]，mes-ingest 已经为此
   自己写过 `PrintWindow` [F5]，选 FlaUI 等于把这段再写一遍。
5. `AutomationId` 是首选选择器 [W8]，与 L2 现有的定位约定一致；WPF 的 `x:Name` 回退 [X1] 让
   `ScanTextBox` 直接命中。

要接受的代价，以及 lab 里怎么处理：

- **Public Preview** [W12]。锁定版本（`v0.6.0`）并把 zip 的 sha256 记进登记册，升级是一次显式决定。
  `main` 的文档与发布版可能不同，写场景时对照标签下的文档。
- **遥测默认开** [W13]。计划任务的动作里设 `WINAPP_CLI_TELEMETRY_OPTOUT=1`，真车上位机不该往外发东西。
- **库包只出 `net10.0-windows`** [W10]，进不了 `net8.0` 工程。这对「当 exe 用」没影响，但排除了
  「把 winapp 当库嵌进 L2 或 lab 的 .NET 代码」这条路。
- **语义 slug 会过期** [W8]。lab 的场景只用 `AutomationId` 与 Name，不把 slug 写进场景文件；
  给两个按钮补 `AutomationId` 走 `w2g/*` 分支的 PR。
- **PowerShell `&&` 与原生 CLI 的 stderr 会卡** [W13 旁注]。包装脚本用 `;` 或显式检查 `$LASTEXITCODE`。

lab 仍要写一层薄的 pwsh 包装（不是 `lab-ui`）：投递 exe、按 [M3] 的手法在 session 1 拉起并把
`--json` 重定向到文件、轮询任务状态、读回 JSON。这层与驱动方式无关，选哪一条都得有。

**`UIAutomationClient` 保持现状，只留在 L2。**它零安装 [U1]、已验证 [U2]，但没有 `inspect`、
`screenshot`、`wait-for`、JSON [U3]，是过时 API [U4]。L2 不必迁移；lab 不以它为动词层。

**FlaUI 是备选**：`winapp ui` 若在实测里（下一节）暴露出阻塞问题，再走这条路。届时 `lab-ui` 必须覆盖：

| `lab-ui` 要自己做的 | 原因 |
| --- | --- |
| JSON 输出契约、退出码约定、错误码（对齐 winapp 的 `element_not_found` / `no_interactive_desktop` 一类） | FlaUI 是库，没有 CLI 层 |
| `inspect`：树导出、深度限制、`AutomationId`/Name/ControlType/IsEnabled/BoundingRectangle | 搬 [F2] |
| `wait-for`：元素出现/消失、属性等于/包含、超时 | 用 [F3] 的 `Retry`，默认 1 s 要改 |
| `screenshot`：`PrintWindow` + `PW_RENDERFULLCONTENT`，不用 `Capture.Element` | [F4][F5] |
| 桌面门禁：`OpenInputDesktop` 判断，注入动词在无交互桌面时拒绝 | FlaUI 不判断 [F6]，mes-ingest 已有代码 |
| 主窗口识别：按必需控件找，不用 `GetMainWindow` | L2.psm1 第 440-446 行 |
| 目标机投递：`dotnet publish -r win-x64 --self-contained -p:PublishSingleFile=true`，体积待测 | 零安装要求 |
| 目标框架 `net8.0-windows`、包版本进 `Directory.Packages.props` | ADR 0056 |

## 未能核实的点

按重要性排序，每条写明怎么核实。都要在 VM 或真车上跑，本次调研没有碰机器。

1. **`winapp ui` 的模式动词在锁屏下对 `SQCD.Agv.Wpf` 是否真的可用。**文档承诺 [W3] 是通用的，
   没有针对 WPF 或锁屏的测试记录。核实：在 `win11-01` 起客户端，`Win+L` 后经计划任务跑
   `set-value` / `invoke` / `wait-for` / `screenshot --json`，看返回与截图内容。
2. **锁屏时 WGC 截图截到的是窗口内容还是黑图。**WGC 文档 [M4] 只讲版本要求，没讲桌面状态；
   winapp 文档说 "WGC unavailable on this system/session" 时退化到 `PrintWindow`（第 288 行），
   没说锁屏算不算。核实同上。
3. **`winappcli-x64.zip` 解开是不是只有一个 exe。**csproj [W1] 说单文件自包含，但没打开过 zip。
   核实：下载后列目录，记体积与 sha256。
4. **从 session 0 用计划任务跑 `winapp ui … --json > file`，退出码能否经 `LastTaskResult` 读到。**
   [M3] 的文档只说返回运行时信息，没说 `LastTaskResult` 是否等于进程退出码。核实：跑一个必定
   超时的 `wait-for`，看 `LastTaskResult` 是不是 1。
5. **托管 UIA2 与 FlaUI 在锁屏下的行为。**两者都没有文档承诺，只有在选了备选方案时才需要测。
6. **pwsh 7.6.5 直接 `Add-Type -Path` 加载 FlaUI 的 `net8.0-windows` dll 是否可行。**这会让
   FlaUI 也变成「拷三个 dll」的零安装路径，省掉 `lab-ui` 的发布。未查证 .NET 10 宿主加载
   `net8.0-windows` 程序集与 `Interop.UIAutomationClient` 的兼容性。
7. **真车上位机的 `HT` 账户是本地管理员且空密码**（`remote-ops/onboard-hmi/README.md` 第 61 行）。
   计划任务以该账户 Interactive 登录类型运行是否受「空密码账户限制」策略影响，10-start 脚本
   已经在真车上跑通过，所以大概率没问题，但那是起客户端，不是起驱动程序。
