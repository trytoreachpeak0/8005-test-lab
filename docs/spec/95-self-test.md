<!-- 来源：#29 #16 -->

# lab 自己怎么测自己

`tests/` 的三分目录、Pester 5、以及「真车上也跑，不加额外限制」由
[#16](https://github.com/trytoreachpeak0/8005-test-lab/issues/16) 定，见
[90-repository-layout.md](90-repository-layout.md) 第 6 节。本文定**测什么、测到什么程度、
哪些不测**。CI 怎么跑归 [#13](https://github.com/trytoreachpeak0/8005-test-lab/issues/13)。

## 1. 什么必须有测试：坏掉的时候会不会不出声

判据与 `tools/toolkit.json` 的 `upgradeRequiresFieldTest` 同一条（#16 定）：
**这段代码坏掉的时候会不会不出声。** 不是「它重不重要」，也不是覆盖率。

| 内核的一块 | 坏了 | 覆盖 |
| --- | --- | --- |
| 谓词求值器与四值 | 不出声 | **必测**，且必须覆盖它静默的那条路径 |
| 时间算子（`stableFor`、`olderThan`、ISO 8601 时长解析） | 不出声 | **必测** |
| 集合算子的比较键 | 不出声 | **必测** |
| 安全门四值与 Grant 到期 | 不出声 | **必测** |
| 证据封存、`completeness.json`、`unsealed` 判定 | 不出声 | **必测** |
| 重连判定（`remoteStateIndeterminate`、120 s 累计） | 不出声 | **必测** |
| 模块依赖方向（#16 那两条） | 不出声 | **必测**，见第 2 节 |
| 求值器只有一个入口（回放与采样环共用） | 不出声 | **必测**，见第 2 节 |
| 绑定校验、场景清单静态校验 | 响 | 不强制 |
| NDJSON 分帧、`workRoot` 管理 | 响 | 不强制 |
| CLI 参数解析、JSON 信封 | 响 | 不强制 |
| 适配层加载与 schema 校验 | 响 | 不强制 |

「不强制」不是「禁止」：这些地方没有测试不算欠债，写了也不算浪费。

**每个 `tests/unit/` 的 `Describe` 写一句它锁的是哪一种静默**，同 `toolkit.json` 给每个工具写一句
`upgradeRequiresFieldTestReason`。

**不设覆盖率百分比门槛。**

## 2. 架构测试：两条约束加它们自己的锚点

扫四个 psd1 的 `RequiredModules` / `NestedModules` 与四个 psm1 的 `Import-Module` / `using module`：

- `Lab.Judge` 不依赖别的三个模块，也不做 I/O；
- `Lab.Remote` 与 `Lab.Probe` 不依赖 `Lab.Core`；
- `Lab.Judge` 的求值器只有一个入口 —— 回放与采样环都从那里进（#21：「回放不是第二套引擎」）。

**锚点自检是必需的一半**：断言扫到的模块数等于 `src/` 下的目录数。没有它，改一个目录名就会让这个
闸门静默地什么都不扫、然后一直绿。做法取自被测仓的
`SQCD.Agv.UnitTests/ReasonCodeRegistryArchitectureTests.cs`。

## 3. `tests/machine/`：Footprint 声明既是足迹也是选择器

每个 `tests/machine/` 的测试声明一个 `$LabTestFootprint`，**没有声明即加载期拒绝**
（`machineTestFootprintMissing`）：

```powershell
$LabTestFootprint = @{
    Requires = @('interactiveDesktop')   # 词汇取 registry.json 的机器能力字段（#5）
    Touches  = @{
        Processes = @('pwsh', 'winapp')
        Ports     = @()
        Desktop   = $true
        Paths     = @('<workRoot>/selftest/<runId>')
    }
}
```

**一份声明两个用途**：`Touches` 交 [#31](https://github.com/trytoreachpeak0/8005-test-lab/issues/31)
的足迹核对；`Requires` 派生出「这台机器上跑哪些」。**不另写 tag 数组** —— 两份清单必然分叉，
而分叉是静默的。

### 3.1 够不着的机器：`notAttempted`，不是绿也不是红

一次自测产出 `coverage-attempted.json`，按 `Requires` 逐组列：要什么、哪些机器满足、实际跑了
哪几台、哪几台够不着（`machineUnreachable`）。

**够不着不判红**（与 #11「回捞的失败不改 run 结局」同向），**但必须可见** —— 它是
`notAttempted`，与 #21 的 `notApplicable`（这套装置下不适用）是两个值。

### 3.2 那批「只有在真机器上才成立」的，逐条

| 东西 | 在哪 | 怎么测 |
| --- | --- | --- |
| 重连的判定逻辑 | `tests/unit/` | 纯函数：断连时长 + `sideEffect` + 远端状态读回 → 重放 / 放弃 / `remoteStateIndeterminate` |
| ssh 管道在断点上的行为 | `tests/machine/` | 在控制端到 `vm01` 的管道上制造：kill ssh 进程、关读端、远端 shell 退出。**不动虚拟交换机** |
| CPE 切 AP 那几秒的时序 | **不测** | 那是观测不是断言（#23：一旦 lab 去驱动它，那个数就是编出来的） |
| 桌面互斥体 | `tests/machine/` | 退出码 3、`$LASTEXITCODE` 先清、持有者被杀后下一次 `WaitOne(0)` 为真 |
| `AbandonedMutexException` 那条路 | **不强制** | 要同时起一个持有者和一个阻塞的等待者再杀掉前者；先例（`Invoke-WithDesktopLock.ps1`）自己也没覆盖它 |
| 证据封存与 `unsealed` | `tests/unit/` | 写 `MANIFEST.sha256` 之前 kill，控制端就能做，**不需要真机器** |
| 时钟事实的来源 | `tests/machine/` | 钉住：取自 `clock.json`，不取自会话登录时间或 `LastBootUpTime`（`agv01` 的 console 会话至今报 2027/2/5） |

## 4. 驱动 `winapp ui` 的那层薄包装：三层，最要紧的一层不是测试

| 层 | 在哪 | 测什么 | 要什么 |
| --- | --- | --- | --- |
| 命令行构造 | `tests/unit/` | 参数里**永远不含 `--capture-screen`**；只出现读侧动词；截图硬编码 WGC；`WINAPP_CLI_TELEMETRY_OPTOUT=1` 设进程环境 | — |
| 真的驱动一个窗口 | `tests/machine/` | 起夹具 → 按 `AutomationId` 命中 → 读值 → 截图判据在一张真图上跑得出结论 | 一个交互式桌面 |
| **锁屏下的三条行为** | **不是测试** | — | 人跑的实测，`upgradeRequiresFieldTest: true` 触发，落点归 [#36](https://github.com/trytoreachpeak0/8005-test-lab/issues/36) |

第三层测不了：要验它就得真的锁住一台机器的桌面，而锁完之后跑测试的那个会话自己也在那块桌面上。
**既然真实行为测不了，能被机器锁住的就只剩契约本身** —— 这是第一层存在的理由。

### 4.1 夹具是 pwsh 起的 WPF 窗口

`Add-Type -AssemblyName PresentationFramework` 加 `XamlReader.Load`，放 `tests/fixtures/`。
实测（2026-09-08，控制端）：另一个进程用 `UIAutomationClient` **按 `AutomationId` 与按 Name 都
命中**，`ControlType.Edit` 正确。**不需要 .NET 工程。**

夹具刻意长成被测系统的形状：一个 `AutomationId = ScanTextBox` 的 `TextBox`，一个只能按 Name 匹配
的中文按钮 —— #7 的两条结论就是在这两种控件上得出的。

**不设 `Topmost`、`ShowActivated="False"`、标题带 `runId`、生命周期以秒计。**

### 4.2 跑在哪

控制端、`vm01`、三台车都行。**不要求黄金渲染机那七条环境条件**（分辨率 / DPI / 主题 / 字体 /
locale / 时区 / 渲染模式）—— 那七条是为像素基线服务的，这里不产像素基线。

`tools/toolkit.json` 的 `winapp-cli.machines` 因此含 `local`。

**生产车上无条件跑**（用户 2026-09-08 定）。#15 那条「`tier: production` 上 UI 写操作一律无条件
拒绝」不覆盖它 —— 那条禁的是对**被测系统窗口**的写操作。

**在 `vm01` 上跑这一组必须拿 `Global\W2G-InteractiveDesktop`**，lab 是那台机器上的第五个持锁者。

## 5. 回放与单元测试的分工

| 东西 | 在哪 | 是什么 | 它绿了说明什么 |
| --- | --- | --- | --- |
| 谓词求值器在边界输入下的行为 | `tests/unit/` | 测试 | 引擎在畸形输入下不乱来 |
| 回放器能读一个证据目录并跑完 | `tests/unit/`（吃 `tests/fixtures/` 的最小素材） | 测试 | 回放这条路本身通 |
| 规则集在历史素材上回放 | **不在 `tests/` 里** | **交付物**（一次 `lab invariant replay`，产 Evidence） | 规则集在这批历史故障上命中 |

**回放不进 `tests/`。** 放进去，「回放跑绿了」和「引擎测过了」就共用同一个绿。同 RC 第 12 节
（「八类 G3 向量各有证据不等于八个切片通过」）与 `golden-renderer.md`（narrowed run 记
`WATCH_UI_PARTIAL_PASSED`，永远不是 `WATCH_UI_ALL_PASSED`）。

**边界输入那批最该覆盖，因为它们在回放里永远不出现** —— 空集合、null 字段、类型对不上、畸形
时长，真实素材里一份都没有。反过来，边界输入的单测也永远说不出「这条规则在真实故障上不命中」。

**`tests/fixtures/` 的最小素材是造的，不是从历史证据里挑的**（真素材不含畸形输入，且一份证据
目录带着 `logs/` 与 `snapshots/`）。真素材的回放读 `evidence/l2/` 原地。

### 5.1 `--legacy-l2` 必须报告它不认识的 criterion

`criterionNotMapped` 是一个 `reasonCode`，**不等于 `notApplicable`**：后者是「素材天生不支持」
（#21 那七个 hashtable 渲染的），前者是「素材可能支持，只是没人写映射」。

不是可选的诊断，是回放的一部分。理由是数出来的：#21 写下那份 34 个 criterion 的映射当天下午，
新素材就带来了第 35 个（`loaded-demands`），而它是纯标量、**可反解**。

### 5.2 验收分母：规则是权威，快照是派生

**规则**：`outcome` 非 `PASS`，且失败不是 lab / 编排自身的异常（`failureReason` 是一条 pwsh 运行时
异常而不是一次断言比对）。**分不出类的报错，不默认归入任一边。**

**快照**：`tests/fixtures/legacy-l2/acceptance.json` —— 目录名 + sha256 + 分类 + 一句理由。验收时
重算并 diff。

规则解决腐烂，快照解决复现。2026-09-08 17:30 按此规则算得 **18 份**（100 份总数、21 份非 PASS、
减 3 份工具自身异常）；同日 02:43 是 16 份（74 份时代）—— 这个数字会继续变，所以它不是标准，
规则才是。

## 6. 自测产物

**不是 Evidence，不进 `evidence/`。** 一次自测没有 scenario、没有 rig、没有角色绑定。

落 `<workRoot>/selftest/<runId>/`，**跑完即清、只回传结论**（Pester 结果与
`coverage-attempted.json`）—— 跟探针那一类，不跟远端半边那一类。

## 7. 测试代码送多少上车

`tests/unit/` 与 `tests/machine/` 跟着远端半边走同一条内容哈希同步。**`tests/fixtures/` 显式
排除**，且排除是同步清单里一条**带理由的具名项**，不是一个不在列表里的目录。

- `unit/` 也上车，因为 pwsh 的字符串比较、日期解析、数字格式化受 culture 影响，而车上是 zh-CN
  —— 时长解析与时间戳比较正是第 1 节「不出声」的那一批。
- `fixtures/` 不上车，因为回放在控制端跑（`Lab.Core` 只在控制端），素材对车没有意义。

与 RC 第 12 节「发布包里只有可运行的二进制、不含测试宿主」方向相反，刻意的：#16 的模块分界是
「在哪台机器上加载」，要验的就是它在那台机器上加载起来之后的行为。

## 8. 与 `adapters/example/` 的边界

`tests/fixtures/` 里的假 adapter 与 `adapters/example/`（[#19](https://github.com/trytoreachpeak0/8005-test-lab/issues/19)）
**是两样东西，不合并**。判据是读者不同：前者的读者是内核的测试，它应该长得畸形（故意缺字段、
故意声明错的 `contractVersion`）；后者的读者是一个要接入 lab 的外部系统，它必须长得像一个好例子。
