<!-- 来源：#16 收拢子命令，#33 定形状 -->

# CLI

agent 与 lab 之间只有这一条接口。子命令名收拢于 #16，参数、输出、退出码与 `reasonCode` 的命名空间
定于 #33。**票是权威，这份是派生。**

入口是仓库根的 `lab.ps1`：`pwsh <仓库>/lab.ps1 <子命令> ...`。不装 PATH shim。

## 参数风格

**面向 agent 的参数一律 kebab-case 双破折号**（`--evidence-root`、`--bind`、`--legacy-l2`）。

pwsh 本身不认双破折号 —— 实测直接报 `A positional parameter cannot be found that accepts argument
'--evidence-root'`。所以 `lab.ps1` 用 `[Parameter(ValueFromRemainingArguments)]` 收下原始参数，
自己把 `--kebab-case` 转成 `-PascalCase` 再 splat 给子命令脚本。

**`lab help --json` 的派生器必须把名字转回 kebab 再输出。** 不转的话文档写一套、help 报另一套，
而 agent 照 PascalCase 敲 `-EvidenceRoot` 居然也能跑通（pwsh 原生绑定），于是没有任何东西会报错。

布尔参数一律 `[switch]`，不用 `-Foo $true`。`ValidateSet` 的取值全集就是 agent 看到的 enum。

## 全局参数

放在子命令**之前**。判据是「它是不是 lab 这次运行的坐标系」。

| 全局参数 | 默认 | 定于 |
| --- | --- | --- |
| `--registry <file>` | 仓库根 `registry.json` | #5 |
| `--evidence-root <dir>` | 仓库 `evidence/` | #11 |
| `--observation-root <dir>` | 仓库 `observations/` | #20 |
| `--adapter <name>` | 无默认，只有一个适配层时可省 | #18 |

其余一律子命令级，**包括 `--bind <role>/<instance>=<machine>` 与 `--bindings <file>`** —— 它们是
这一次 run 的阵容，不是坐标系。`lab watch start` 的 `--courier <file>`（#28）与 `--vigil <file>`
（#30）同样是子命令级：它们是这一段 watch 的通路，不是 lab 的坐标系。两者都**给路径不给 URL**
（URL 写在命令行上会进进程列表与 shell 历史），文件都进 `.gitignore`。

**没有配置文件，也不读环境变量。** 一条命令就是它自己的完整输入：#12 已经在配置改写上立过同一条
规矩（整份渲染 + run 后还原 + production 上记 sha256），再开一个「值从哪来」的层级等于把那条规矩的
反面引进 lab 自己，而它失败的方式是静默的 —— 同一条命令在两台机器上结果不同，证据里看不出为什么。
环境变量另有一条：#16 已经因为足迹把 `WINAPP_CLI_TELEMETRY_OPTOUT` 定成每次调用的进程环境、
不设机器级。

## 子命令

| 子命令 | 定于 |
| --- | --- |
| `lab run <scenario>` | #6 #10 |
| `lab watch start` / `stop` / `status` / `grant` / `mute` / `unmute` / `courier-check` / `vigil-check` | #20 #15 #28 #30 |
| `lab scenario list` | #10 #22 |
| `lab machine check` | #5 #6 #12 |
| `lab adapter check` | #4 |
| `lab do <role> <verb> --verb-args <file>` | #4 #6 |
| `lab capture verify` / `seal` | #11 #20 |
| `lab observation render-ledger` | #22 |
| `lab invariant list` / `check` / `replay` | #21 |
| `lab read <role>[/<instance>] <controlPlane\|tap>` | #6（原 `lab probe`，#16 改名） |
| `lab toolkit sync` / `verify` | #16 |
| `lab help --json` | #33 |

`lab do` 的动词参数**只有 `--verb-args <file>` 一种形式**，不做 `--verb-arg k=v`：#6 已经定了到
远程那一跳「参数走 JSON 文件不走命令行」，CLI 这一侧收同一份文件就不需要一个只在控制端存在的中间
形态。两种形式并存必然分叉，而分叉的那天是「同一个动词从命令行调和从场景里调，参数解析不一样」。

`lab watch status --all-machines` 的形状定于 #33（是一个开关，不是新子命令）；**要不要有这条能力
归 #34**。

**明确不做**：`lab session open`（#6 拒绝，理由未变）。

**留给别的票**：`lab capture list` / `find`（#25）、`lab observation prune`（#26）。

## 输出：两个信封

分界是**一次性结果**与**流式**。`lab watch start` 在前台是长命且持续产出的（#20），一行 JSON
装不下它，而它要产的正好是 lab 在证据里已经在用的 NDJSON。

### Result 信封

stdout 一行 JSON，一次性子命令用：

```json
{"lab":1,"command":"run","outcome":"PASS","reasonCode":null,"adapterReasonCode":null,
 "startedAt":"2026-09-08T10:00:00.000Z","endedAt":"2026-09-08T10:04:12.331Z","data":{}}
```

- `lab` 是信封版本（整数，今天是 `1`）。**不叫 `schemaVersion` 也不叫 `envelopeVersion`** ——
  `Envelope` 与 `schema` 在被测系统里都已经被占（`WireToGateEnvelope` 62 处、
  `PROTOCOL_ENVELOPE_INVALID`、`PROTOCOL_SCHEMA_INVALID`）。
- 时间字段 UTC ISO-8601 带毫秒。
- `outcome` 的取值全集按子命令定；**`lab invariant replay` 没有这个字段**，见下。

### Stream 信封

stdout 每行一个 JSON。**每行自带 `kind`** —— 流的消费者要能在不知道自己在读哪条命令的情况下分帧后
分类。**最后一行 `kind` 恒为 `result`，内容就是 Result 信封**，所以一条流读到底等于读到一次一次性
结果，agent 不需要两套解析。

行尾写 `\n`，**读的一方必须容忍 `\r\n`** —— pwsh 在 Windows 上默认写 CRLF（实测）。

**中断**：`lab watch start` 前台被 Ctrl+C 或杀掉时，流的最后一行应当仍是 `kind: result`、退出码 0
（人为中断是正常结束，`data` 里记 `stoppedBy: interrupt`）。若实现时发现 pwsh 在中断路径上保证不了
这一行，不硬撑，改用这条同样可核的规则：**agent 判断一条流是否完整收尾，看它的最后一行是不是
`kind: result`** —— 不是就说明被截断了，去机器上捞探针的账（探针那侧按 #20 已定的自杀并留
`probe-exit.json`）。

### 失败也走 stdout 的同一个信封

**这与 #4 不同，而且这个不同是有意的。** #4 定的「成功 stdout 一行 JSON、失败 stderr 一行带
`reasonCode`」是**内核与适配层之间**的约定，两侧都是 lab 的代码，把失败分流到 stderr 是安全的。
CLI 与 agent 之间不一样：agent 要的是**一次读取拿到完整结论**，把结论劈到两个流上等于让它去猜哪个
流先到、要不要等另一个。

所以 CLI 这一侧 **stdout 永远是结论，stderr 只有两样东西**：人读的诊断（含 `--help`），和 pwsh
自己吐的错误。

### stdout 纯净靠结构，不靠扫描器

**实测：子进程 stdout 被重定向时，`Write-Host` / `Write-Warning` / `Write-Verbose` 全都落进
stdout**，只有 `Write-Error` 与 `[Console]::Error.WriteLine()` 进 stderr。所以这条不是一句声明就
成立的事。

内核只有两个输出出口 —— `Write-LabResult`（stdout）与 `Write-LabNote`（`[Console]::Error`）；
**`src/` 下不出现裸的 `Write-Host` / `Write-Warning` / `Write-Verbose`**。

不加扫描器的理由是 `95-self-test.md` 那条判据：**这条坏掉的时候是响的** —— 有人写了一句
`Write-Host`，agent 下一次读到的 stdout 当场解析失败。每个子命令的自测顺带断言一次「stdout 整体
可解析」即可，那是「响的，不强制」那半里顺手做的一条，不是新增一项强制。

### `lab observation render-ledger` 是唯一的例外

它的产物是一份 Markdown 文件（#22）。stdout 仍是 Result 信封，`data` 里给出写到哪、渲染了多少条、
覆盖的 UTC 日期范围；**账本身不进 stdout** —— `ledger.md` 是渲染产物，权威数据始终是 jsonl。

## 退出码

切法是 `95-self-test.md` 那条判据的第三次使用：**这一刀分错了会不会不出声。**

| 码 | 含义 | CI 该怎么读 |
| --- | --- | --- |
| 0 | 该命令成功完成（`run` 的 `PASS`） | 绿 |
| 1 | **保留不用** | lab 自己崩了，崩在它自己的错误处理之前 |
| 2 | 判定失败（`run` 的 `FAIL`） | 红 |
| 3 | 工具自己出错（`run` 的 `ERROR`） | 红 |
| 4 | 没跑成（`notAttempted`） | **不算红，但绝不算绿** |
| 5–7 | 保留 | —— |

**1 保留不用**，理由是实测：pwsh 里未捕获的 `throw` 与语法错误都退 1，而 `Write-Error` 退 0。
把 1 留空，「lab 在它自己的错误处理之前就崩了」这条信息就免费拿到了 —— 否则它与场景判定失败共用
一个码，那正是 `Invoke-L2Scenario.ps1:582` 那条 `if ($outcome -ne 'PASS') { exit 1 }` 今天的处境。

**4 是 `Invoke-WithDesktopLock.ps1:93` 那条 `exit 3`（`WATCH_UI_SERIALIZATION_BUSY`）的推广** ——
「不是失败，是没跑成」。#29 已经用 `notAttempted` 把这一刀定进了 `tests/machine`，这里让它到达
shell。今天 `agv02` / `agv03` 就够不着。

**「能不能重跑」不上退出码**，挂在 `reasonCode` 的 `retryDisposition` 上。#10 那条「`ERROR` 可重跑
但带 `remoteStateIndeterminate` 的不行」由此变成一次查表。理由仍是那条判据：重试这一刀分错了是响的
（CI 重试了一个不该重的，下一次运行当场暴露），而「没跑成」这一刀分错了是不出声的。

**安全门的四值（#15）与判据的四值（#21）不上退出码** —— 一次 run 里有很多条门决定和很多条判据结果，
退出码只有一个。它们在 JSON 里，`gate-decisions.jsonl` 与断言结果里已经各有各的位置。

**取值只用 0–7。** Windows 上 pwsh 的退出码是 32 位（实测 `exit 300` 原样传出 300），但 POSIX 那侧
会截断到 8 位，lab 没有理由去消费那个空间。

**加新码要过同一道判据**：只有当一个新的区分「分错了会不会不出声」答案是「不出声」时，它才配一个
新退出码；答案是「响的」就进 JSON。

## `reasonCode`

**唯一定义处是 `src/Lab.Judge/reason-codes.json`**，形状照
`8005-agv-protocol/errors/error-codes.json`：

```json
{ "registryVersion": "1.0.0", "appendOnly": true, "displayMessageAuthoritative": false,
  "codes": [
    { "code": "remoteStateIndeterminate", "layer": "kernel", "definedBy": "#10",
      "retryDisposition": "MANUAL_REVIEW", "meaning": "...", "introducedOn": "2026-09-08" }
  ] }
```

放 `Lab.Judge` 而不是 `Lab.Core`，理由是 #16 那条分界：模块的界是「在哪台机器上加载」。`Lab.Judge`
纯函数无 I/O、控制端与探针两处都加载，而 reasonCode 在探针上也要发得出来（离线求值、失明期照修）。

三条照抄协议仓：

- **`appendOnly: true`** —— 码发布后不换义、不复用（同 `mes-ingest` ADR 0011 对 `SeriesErrorCatalog`
  定的）。
- **`displayMessageAuthoritative: false`** —— **码是权威，文本会骗人**。活证据是
  `RELEASE-CANDIDATE.md` 第 8.1 节那张实测对照表：车载端的逐字文本「ControlServer 在会话恢复期间
  关闭了连接」听起来像业务层的恢复问题，与真因（两端 TLS 期与明文期错配）毫无关系。
- **`retryDisposition`** —— 收窄成 lab 需要的四个：`NEVER` / `AFTER_STATE_CHANGE` /
  `AFTER_RECONNECT` / `MANUAL_REVIEW`。

`definedBy` 记票号不记文件路径 —— 票是权威（#16）。

### 守法靠结构，扫描器只当锚点

内核代码里**不写码字面量**，只引 `Lab.Judge` 导出的查表函数（#21「求值器单一入口」在另一个对象上的
同一条）。架构测试扫 `src/`，两条断言：

1. 扫到的码字面量不在表里 → 红。
2. **锚点**：扫到的、标 `layer: kernel` 的码数 ≥ 表里 `layer: kernel` 的条数，否则改个写法它就静默
   失明（#29 定的「架构测试自带锚点」）。

做法取自 `ReasonCodeRegistryArchitectureTests.cs`，连它两条纪律一起带过来：**扫描不按文件名过滤**
（「a reason code must not escape the gate merely by living somewhere else」），以及**本地码命中
规则时加显式豁免并写理由，而不是收窄规则**（收窄会静默丢掉真码的覆盖）。

这一条**属于 `95-self-test.md` 必测那半**：两个地方定义同一个 `transportUnstable`、含义不同，
agent 照读照做决定，没有任何东西会报错。

### 内核的码与适配层的码是两个字段

- **`reasonCode`** —— lab 自己怎么了：传输、安全门、判据、工具链。表在 `Lab.Judge`。
- **`adapterReasonCode`** —— 被测系统怎么了：#4 定的，适配层自己定义并写进接入文档。

两个字段永远不会撞，**所以命名空间隔离这件事不存在**；#18 只需要定多张适配层码表怎么加载与卸载。
选它的理由是**「谁该修」在信封上就读得出来**：`reasonCode: sshUnreachable` 是 lab 的事，
`adapterReasonCode: LOCK_NOT_CLOSED` 是被测系统的事，agent 不必查表就知道去哪一侧。

代价：**有些失败同时属于两边**（钩子超时 —— 传输没断，是适配层的钩子没回）。规则是两个字段
**可以同时有值**，只有一边成立时另一边是 `null`。

## `lab invariant replay` 的输出

**结局词不是通过/失败，而且不共用字段名。** 它的信封里**没有 `outcome`**，有的是 `replayVerdict`：

| 值 | 含义 |
| --- | --- |
| `hit` | 命中已知故障 |
| `miss` | 没命中 |
| `criterionNotMapped` | **映射表不认识这个 criterion**，与 `notApplicable` 是两回事 |

`notApplicable`（#21）是「素材天生不支持」，由 `AppliesTo` 在加载期判掉；`criterionNotMapped` 是
「素材可能完全支持，只是没人写映射」—— 那份 34 个 criterion 的映射表在写下的当天下午就漏了第 35 个
（`loaded-demands`），而它是可反解的。

在字段名上就分开，做法取自 `golden-renderer.md`：narrowed run 记 `WATCH_UI_PARTIAL_PASSED`，
**永远不是** `WATCH_UI_ALL_PASSED`，而且写的是另一个文件名 —— 不指望读的人记得区别。同一句话在
`RELEASE-CANDIDATE.md` 第 12 节：八类 G3 向量各有证据不等于八个切片通过，切片仍是 `INCONCLUSIVE`。

**回放永远退 0**（除非 lab 自己出错，那是码 3）。`miss` 是一个**发现**不是一次失败 —— 回放产
Evidence、是 destination 第 2 条的交付物，不是测试（#29 定的「回放不进 `tests/`」）。让它退非 0，
「回放跑绿了」就会重新与「引擎测过了」共用同一个绿，只是换到了退出码上。

## `lab help --json`：给 agent 读的那一份

**现场从 `param` 块派生，不落盘进 git。**

实测：`(Get-Command <script>).Parameters` 给出名字、类型、是否必填、位置、`ValidateSet` 取值全集，
一次 `ConvertTo-Json` 就是完整的一份 —— 不需要手写清单，不需要 .NET 工程。

**不落盘是这条决定的全部要点。** `Get-WpfDesktopTestFilter.ps1` 的注释写着「a hard-coded list is a
list someone will forget to update」，**而它第一版就漏了两个类**。一份落盘的命令表就是那份手写清单。

它给出三样：

1. 子命令与参数（**派生**，名字转回 kebab-case）。
2. 每个子命令的**结局取值全集** —— `run` 是 `PASS`/`FAIL`/`ERROR`，`machine check` 含
   `notAttempted`，`invariant replay` 是上面那三个。这是唯一无法从 `param` 块派生的部分，所以它是
   手写的那一小块，跟着各子命令的实现走。
3. 退出码表。

派生的坑：**无位置参数的 `Position` 是 `-2147483648`**，派生器要归一成 `null`。

**不写每个子命令的 JSON schema。** 判据是 `95-self-test.md` 的：schema 校验坏了是响的。十一个子命令
连动词二十几份 schema 是一份真实的工时，而它买到的东西 agent 靠解析失败就能发现。

`--help`（人读的）保留，**走 stderr** —— 它是给人的诊断，不是结论。

## 两条已定的命名事实

- **`lab probe` 改名 `lab read`**：与 #20 的 Probe（探针）撞名。没选 `inspect`（#7 占，是内核 `ui`
  控制面暴露的 UIA 模式动词之一）也没选 `snapshot`（#4 占，是可选的生命周期钩子名）。`read` 还多说
  对了一件事——它只有读侧，正好覆盖 #23 定的 Tap。
- **`lab machine check` 与 `lab toolkit verify` 的分界**：前者核登记册的声明对不对，后者核 lab 自己
  放在那台机器上的东西还在不在、哈希对不对。Toolkit 不在登记册里。

**#33 不造新词**，`CONTEXT.md` 因此不动：`Envelope` 与 `Catalog` 在被测系统里都重度占用
（`WireToGateEnvelope` 62 处、`CatalogRevision` 54 处），而 `Registry` 这个词条已经归机器登记册
（#5）。「Result 信封 / Stream 信封」是这份规格的局部说法，不是内核词汇表的词条。

## 附录：`reasonCode` 快照（2026-09-08）

**规则是权威、快照是派生。** 下面这份是截至 2026-09-09 已在票里点名的码，**至少 50 个**（#28 关闭时
加了第 49 个，#30 加了第 50 个），只用来给
`reason-codes.json` 的第一版打底 —— **它会长**，长了不必回来改这份附录，改注册表。

**「至少」是字面意思，不是谦辞。** 这份是从三十多张票的正文与评论里按「反引号包着的 camelCase
标识符」筛出来再逐条核对语境得到的，这个筛法必然漏 —— 第一轮就把 `violationAbandoned` 漏在了名单
外（它在 #22 那张「`reasonCode` / `decision` / 说明」表里），而反过来 `declaredBuildCommitMatchesBuild`
（RC manifest 的字段）与 `decidedOffline`（账里的布尔字段）差点被误收进来。所以这份是**下界**，
真正的清单以 `reason-codes.json` 为准。

- **#6 传输**：`sshUnreachable`、`sshAuthFailed`、`remoteInvocationFailed`、`hookFailed`、
  `remoteStateIndeterminate`、`transportUnstable`、`transportLost`、`interactiveDesktopNotLoggedOn`、
  `unknownSshAlias`、`machineUnreachable`、`workRootMissing`
- **#12 部署**：`pinNotOnOrigin`、`runtimeMissingOnTarget`、`readbackPinMismatch`、
  `unexpectedRoleRunning`、`preDeployStateUnrecorded`、`roleNeverStarted`
- **#15 安全门与 Grant**：`deniedBySafetyGate`、`deniedByGrant`、`noPreconditionsDeclared`、
  `preconditionNotMet`、`grantExpired`、`grantBudgetExhausted`、`grantLevelInsufficient`、
  `authorityExpired`、`haltFilePresent`、`productionUiWriteForbidden`、
  `productionPhysicalTtlTooLong`、`evaluationBudgetExceeded`、`evaluationTimedOut`、
  `stateUnreadable`
- **#21 判据与不变量**：`evaluationFailed`、`predicateEvaluationFailed`、`comparisonKeyMissing`、
  `invariantIdMalformed`、`fastCadenceUnexplained`、`sourceUnreadable`、`criterionNotMapped`
- **#22 修复与账**：`noCompletionCheckDeclared`、`configureForbiddenInWatch`、
  `violationNotObserved`、`violationAbandoned`、`readOnlyDiagnosticViolated`、`unknownOperator`
- **#23 rig**：`externalRolePinned`、`scenarioUnderdeclared`
- **#16 / #29 工具链与自测**：`pesterVersionTooOld`、`machineTestFootprintMissing`
- **#28 送达**：`courierNotConfigured`
- **#30 守望**：`vigilNotConfigured`

`#33` 正文当时写的是「至少十几个」—— 那是低估，数出来至少 48 个（#28 之后 49 个，#30 之后 50 个），而 #29 一张票就加了 2 个、#21 加了
七八个。这正是**凡是往决议里写一个数就同时写下它是什么时候的、会不会长**那条规则的又一例，而这次
连「数出来的那个数本身」也需要一句它的筛法说明。
