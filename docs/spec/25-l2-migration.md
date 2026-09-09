<!-- 来源：#17 -->

# L2 迁移：今天的 `scripts/l2/` 怎么变成 lab 的第一个场景库

`8005-agv-control-server/scripts/l2/` 今天的 1398 行编排、11 条场景与 102 份证据，怎么变成
`adapters/wire-to-gate/` 的场景库。这份规格既讲一次性的迁移路线，也讲迁移过程中定下的、之后一直
成立的分界。

## 1. 前提：这不是一次搬运

`scripts/l2/` 全部 23 处地址是 `127.0.0.1`，**零处 ssh、零处远程调用**。今天没有任何一条 L2 场景
跨过机器，包括三条 `real-onboard-*`——它们跑真 WPF 加真模拟器，但都在同一台机器上。

destination 第 1 条要的东西分散在两处，而两处完全不相交：

| | `scripts/l2/` | `remote-ops/onboard-hmi/scripts/11,12,13` |
| --- | --- | --- |
| 机器数 | 1 | 2（`agv01` + `factory01`） |
| 断言 | **125 条** | 0 |
| 证据 | 102 份 | 0 |
| 上过真车 | 从未 | 是 |

**迁移的产物是这两半在 lab 里第一次合上**：L2 交出判定，`11` 交出拓扑。

两条从 `11-drive-journey.ps1` 得到、之后一直成立的约束：

- **被测系统的两个自动化面（车载端与模拟器）故意绑 `127.0.0.1`。** 所以触碰它们的步骤必须在那台
  机器上执行。这由 #6 的传输承担：**场景正文仍是控制端上的一个脚本，跨机器的是原语不是脚本。**
- **一个切分点如果承载正确性，它必须是场景正文里显式的一次 `Wait-LabCondition`，不能藏在动词
  内部。** 判例是 `11` 的 step 2（等 `WAITING_OPERATOR`）：它看起来像记账，实际是 0.4 秒与 23 秒
  的分界——2026-09-07 真车上整条 IO 轨迹落在 0.4 s 内，服务端判了 `LOAD_RESULT_REQUIRES_RECOVERY`。
  藏进动词内部之后，别的场景复用那个动词会静默地继承或丢掉这个等待。

## 2. `L2.psm1` 的十六个导出函数怎么分

分界用 #4 的「内核原语不认任何被测进程名」与 #16 的「模块的分界是在哪台机器上加载」。

### 进内核

| 今天 | 去哪 | 备注 |
| --- | --- | --- |
| `New-L2Journal` | `Lab.Core` | `Observe` 改存 JSON 值，不存 `[string]` 渲染串（#21） |
| `Wait-L2Condition` | `Lab.Core`，改名 `Wait-LabCondition`，`-Component` → `-Role` | 加可选 `-AssertionId`/`-AssertionText`（#10） |
| `Assert-L2ComponentAlive` | 判定进 `Lab.Judge`（#21 的 `role-process-alive`），读 stderr 尾巴进 `Lab.Remote` | |
| `New-L2Assertions` | `Lab.Core` | 原样 |
| `Write-L2Evidence` | `Lab.Core` | 只留骨架，见下 |
| `Invoke-L2Query` 的读侧 | `Lab.Core` 的 `.Db` 面 | 纯 ADO.NET 遍历，不认表名 |

`Write-L2Evidence` 今天硬编码的两段中文 caveat 正文（`L2.psm1:717-727`）**出适配层**，成为 #11 定的
必填 `caveat` 字段。

### 进适配层 `adapters/wire-to-gate/`

| 今天 | 去哪 | 为什么不是内核 |
| --- | --- | --- |
| `New-L2Double` / `L2Double` | 适配层门面 | `runId` 作用域、`commandId` 幂等、`expectedRevision`、409 重试五次是 W2G 替身的方言 |
| `Open-L2Database` | `kind: db` 控制面 | 从被测进程自己的 bin 目录加载四个 `SQLitePCLRaw` dll |
| `New-L2PeerStage` | `stage` 钩子 | 「读 JSON、剥 `//` 注释、改字段、写回」是对端两个进程的配置形态 |
| `New-L2OnboardDriver` | `kind: ui` | |
| `Get-L2Journey` | 适配层 | 40 列三表 join，纯 W2G 知识 |
| `Get-L2DeterministicId` | **`shared/`**（#16 第 5 节） | 纯函数入值出值，逐字节复刻 `WireToGateStore.DeterministicGuid` |
| `Wait-L2Iterations` | 切两半 | 「等一个单调不减的数涨够」进内核 `Wait-LabProgress`；「那个数是假 RIoT 的 `mapStationReads`」是适配层按 rig 声明的进度信号（#10） |

### 死掉

| 今天 | 被谁取代 |
| --- | --- |
| `Start-L2Process` | `start` 钩子 + #6 的传输 |
| `Stop-L2Process` | `stop` 钩子，见第 5 节 |
| `Get-L2PeerPublish` | #12（构建源是 origin 上可达的一个 commit） |
| 58405–58413 九个端口常量 | `registry.json` 的 `reservedPorts`（#5） |
| `-Repository` / `-OnboardRepository` / `-SimulatorRepository` | #12 的 pin |
| 「对端仓脏工作树就 abort」 | #12 已明确终结而不是延续 |
| `exit 1` | #33 的四值退出码 |

## 3. 场景清单：`.setup.psd1` 一分为二

今天的 `<名字>.setup.psd1` **不是静态清单，是运行时环境覆盖**：七份里装 `Onboard`（选 rig）、
`ServerSettings`（注入 ControlServer 的环境变量）、`OnboardSeed`、`ClockSkewMs`、`RecoveryResume`。
只有第一项是 `Rig`。而且**四条场景今天没有 psd1**，所以这是 7 份改写加 4 份新建，不是一次改名。

按「内核在启动任何进程之前必须静态知道」这条判据切：

- **`<名字>.scenario.psd1`**——#10 定的四个字段（`Description` / `Rig` / `SideEffect` / `Cast`），
  再加 **`RigProfile`**（一个具名档位，缺省即 rig 的默认档）。内核读，`lab scenario list` 派生自它。
- **rig 配置进 `adapter.json` 的 rig 声明**，场景只能从已声明的具名档位里选一个，不能自由写。

不让场景自由写环境变量的理由：`ServerSettings` 那些值被 ControlServer 在**启动时**读进 `IOptions`，
改一个就得重起一次被测系统，而 #12 已定被测二进制是安装级、跨 run 存活的。收进有限的具名档位之后，
`lab scenario list` 能在**不启动任何东西**的前提下按档位分组，同档的连着跑不用重起。

**档位的说明落 `adapter.json` 同级的 `rigs.md`**，`lab adapter check` 校验每个档位都有一段。
理由是 #10 保留 `.psd1` 不改 JSON 的同一条：注释承重（今天那七份的注释解释了 `departureSafe`
为什么保持 true、`ClockSkewMs` 为什么正好是 100），而 JSON 装不下注释。

今天七份的档位映射：

| 场景 | `RigProfile` |
| --- | --- |
| `sublot-wait-timeout` | `fast-sublot-window` |
| `load-cancelled-before-sublot` | `slow-sublot-window` |
| `auto-charge-endurance` | `auto-charging` |
| `session-established-while-moving` | `moving-at-handshake` |
| `real-onboard-clock-skew` | `skewed-clock` |
| `real-onboard-recovery-entry-missing` | `recovery-enabled` |
| `real-onboard-normal-load` | 默认档 |

**rig 名统一 kebab-case**（#23 交来）：`SyntheticOnboard` → `synthetic`、`RealOnboard` → `real`。
只改 lab 侧，历史证据不动。

**断言 id 改 `<场景名>-NN`**（#10），今天的 `L2-NL-01` / `L2-AC-10` 这类缩写全部换掉。

## 4. 替身工具：一个都不搬

`tools/` 下六个目录里，真正的替身只有三个：

| 目录 | 是什么 | cs 行数 | L1 用例 |
| --- | --- | --- | --- |
| `ControlServer.FakeRiot` | 替身 | 691 | 13 |
| `ControlServer.FakeMesIngest` | 替身 | 377 | **0** |
| `ControlServer.FakeOnboard` | 替身 | 1041 | **0** |
| `ControlServer.ClockSkewProxy` | 线上代理（故障注入） | 306 | 9 |
| `ControlServer.TestDoubles` | 共享库（控制面骨架） | 344 | — |
| `ControlServer.Conformance` | 与 L2 无关（协议清单 SHA256 预检） | 55 | — |

**六个全部留在 `8005-agv-control-server`。** 理由：

1. #16 定了整个 lab 仓库没有 .NET 工程——硬约束，这条自己就够；
2. `FakeRiot` 与 `ClockSkewProxy` 的 22 个 L1 用例在被测仓，而「替身自己坏了会不会不出声」的答案
   是不会——假 RIoT 悄悄答错，场景会绿，绿的是另一件事；
3. #16 第 5 节已定依赖方向：lab 不引用被测仓的东西，反过来。

**在 lab 里它们是 Role，`kind: double`**（#4）。二进制与 ControlServer 走同一条路：#12 的 pin 指向
`8005-agv-control-server` 的一个 origin commit，`publish` 钩子从那个 commit 构建。**不需要第七个仓，
也不需要额外的 pin**——它们和被测系统本来就在同一个 commit 里。

**通则**：一个被测系统的替身是它自己契约的一部分，**归被测系统仓**，跟着它的 commit 走。这样才能
保证替身与真身讲同一版协议。替身与真身的协议一致性由被测仓自己的 L1 守，lab 不加第二道。

## 5. `stop` 钩子必须有 `completionCheck`

今天 `Stop-L2Process` 只有一条路：`Process.Kill($true)`（Windows 上的 `TerminateProcess`）。
实测后果（2026-09-09，控制端 `%TEMP%`）：**202 个 stage 目录、256.6 MB，198 个含
`controlserver.db`，198 个仍带着 `-wal` 与 `-shm`——「有 db 而 `-wal` 已消失」是 0**。
六天 198 次，一次都没有干净停过，而这件事没有出过声。

优雅停止的入口本来就有，**不用改被测系统**：`ControlServer.Host` 是
`builder.Host.UseWindowsService(...)` 加 `await app.RunAsync()`，生产上就是 `factory01` 的一个
Windows 服务，`Stop-Service` 就是那条路。

严格性等级也不用发明，`remote-ops/onboard-hmi/scripts/06-deploy-onboard-hmi.ps1:391` 已经在真车上
跑过完整形状：`CloseMainWindow()` → 等 10 s → 还在才 `Kill()` → 等 10 s → **再查一遍，还在就
`throw`**。

**定：**

1. **`stop` 钩子必填 `completionCheck`**，与 #22 定的动词级 `completionCheck` 共用 #21 的表达形式
   与求值器。缺失即内核拒绝，`reasonCode = noCompletionCheckDeclared`。
2. **不能只判「进程不在了」。** `Kill($true)` 总是让进程不在了，一个只判进程存在性的完成判据在
   198 次里会通过 198 次——它属于「永远不会失败因而永远不出声」的那一类。判据必须落在**被测系统
   的持久状态**上；W2G 这一条是 `controlserver.db-wal` 与 `-shm` 不存在。
3. **`stop` 分两段，`Kill` 是逃生口不是路径。** 先发优雅停止（服务用 `Stop-Service`，GUI 用
   `CloseMainWindow`，控制台用它自己的关停入口），等一个有界的期限，超时才升级。**升级要落进
   证据**：`stopEscalated`（布尔）加 `stopEscalationReason`。一次 `Kill` 收尾的 run 与一次干净停
   的 run 不是同一种证据。

这是 #31 那条「凡是让某样东西不存在的动作必须有 `completionCheck`」在**钩子**上的第一次落地——
#22 定的是动词级的，#4 定的三个必填钩子当时没有这一条。

## 6. 历史证据：不搬

`8005-agv-control-server/evidence/l2/` 今天 102 份（81 PASS / 21 FAIL，其中 1 份尚未提交）。
**一份都不搬**，lab 侧按路径读被测仓。

- **lab 的 `evidence/` 是 gitignore 的**（#16），历史证据在 lab 里没有合法落点；唯一不 gitignore
  的候选 `tests/fixtures/` 被 #29 的「回放不进 `tests/`」排除——它是 destination 第 2 条的交付物
  素材，不是测试夹具。
- **它被 5 个仓的 32 处引用，指向 25 个不同目录**，其中一处是 C# 单元测试
  （`RecoveryStateMachineG2Tests.cs`）的注释。搬动它们，那种断裂不出声。
- 体积不是理由：未压缩 43.7 MiB，**pack 里只占 1.36 MiB**（压缩比 32:1）。真代价是文件数——
  2626 个，占那个仓 71% 的文件，这条落在 #13 的跨仓 checkout 上。

**`--legacy-l2` 回放必须处理的三件事**（#21）：

1. **12 份完全没有 `identity.rig` 字段**（2026-09-03 那批）。回退规则：场景名不以 `real-onboard-`
   开头即 `synthetic`——对今天这 12 份成立。
2. **`real-onboard-resume-after-repair` 的 3 份红证据，其场景今天已不存在**（`d22e9e8` 于
   2026-09-04 改名为 `real-onboard-recovery-entry-missing` 并改了向量）。按场景名对账会找不到。
3. **目录名不可解析，日期前缀不可信**：102 个目录只有 73 个是 `<日>-<场景>-NNN`、6 个是
   `sweepNN`，其余 23 个没有共同形态，其中一个目录名里根本没有场景名；**16 个目录的日期前缀与
   runId 的 UTC 日期对不上**。**场景名与 rig 一律从 `assertions.json` 读，日期一律取 runId，
   目录名只当标识符。**

每份证据是**三个文件加两个目录**：`assertions.json`、`SUMMARY.md`、`timeline.jsonl`，加 `logs/`
（每个组件的 stdout/stderr）与 `snapshots/`（收尾时各控制面与九张数据库表的快照）。体积几乎全在
后两个目录里。

**CI 的证据从来不进 `evidence/l2/`**（`l2.yml` 写 `$RUNNER_TEMP` 再传 artifact），所以这 102 份
全部是手跑的。迁移完成之后这个集合**封顶且不再增长**——交给 #25。

## 7. 迁移路线：逐条切，切一条删一条

一条场景在 lab 里跨机跑绿并封存证据，就从 `scripts/l2/` 删掉。**任一时刻每条场景只存在于一个
地方，不双跑。**

不双跑的理由：今天场景数已经有六个互相矛盾的口径（目录 11、README 表 9、`l2.yml` 数组 8、
地图 6、README 正文 4、`l2.yml` 注释 3），六份没有一份出过声——**没有任何机制在核对它们**。
双跑等于一条场景两份实现，那是同一个形状的第七次。

**六个里有两个是对的**（#13 于 2026-09-09 逐条核过，别把「没有一份出过声」读成「六份都错」）：

| 口径 | 值 | 位置 | 对不对 |
| --- | --- | --- | --- |
| 目录里的 `.ps1` | **11** | `scripts/l2/scenarios/` | **真值** |
| `l2.yml` 的手写数组 | **8** | 第 52–61 行 | **对**（synthetic 确实 8 条） |
| `l2.yml` 注释「the three real-onboard ones」 | 3 | 第 7 行 | **对** |
| `l2.yml` 注释「The synthetic three」 | 3 | 第 18 行 | **错**，实际 8 |
| `README.md` 的场景表 | **9** | 第 19–27 行 | **错**，漏 `multi-demand-one-stop` 与 `load-command-never-answered` |
| `README.md` 正文「现在是四条」 | 4 | 第 64 行 | **错** |

对的那两个正好一个在数组里、一个在注释里。目录里 18 个文件容易误数成 18：那是 11 个 `.ps1` 加
7 个 `.setup.psd1`（**7 带 4 不带**）。

**这三处错的不去改**——#13 判定：它们随 `scripts/l2/` 一起删掉，在一份将要被删的文件上修注释，
收益是迁移期少误导一次，代价是动被测系统仓，而地图 Notes 定死「每张票都以不改被测系统代码为
前提」。**这张表落在这里，因为迁移期读这份文件的人正是会被那三处误导的人。**

不一次性切换的理由：切换点之前 lab 一条不算数、之后 L2 一条不留，中间没有可交付的东西。

### 顺序：先切一条真车的

**`real-onboard-normal-load` 第一个跨机。** 没验证过的东西只在真车那三条上——两个自动化面绑
`127.0.0.1`、CPE 会切 AP、UIA 过不了 session 边界、锁反馈沉降那 0.4 秒的窗口。先把八条合成场景
搬完，只会让内核的形状在碰不到难点的前提下固化。

代价：第一步就是最贵的一步，要 `agv01` 空着、要人在厂区，可能卡很久。

### 一条场景「切完了」的五条判据

1. `<名字>.scenario.psd1` 存在且通过 schema 校验；
2. 角色分布在两台以上机器上跑完，**不接受单机过渡态**；
3. 产出一份 #11 形态的证据，`MANIFEST.sha256` 与 `completeness.json` 齐；
4. `stop` 钩子的 `completionCheck` 通过（第 5 节）；
5. 从 `scripts/l2/scenarios/` 删除，且若它在 `l2.yml` 的手写数组里，**同一次提交里**删掉那一行。

第 5 条那半句防的是漂移：**手写数组与目录的差值只能减不能增。**

### 迁移的完成判据（不是 destination 的）

十一条全部切完、`scripts/l2/` 删除、`l2.yml` 的手写数组删除。

### `scripts/l2/README.md` 那 302 行拆三份

| 内容 | 去哪 |
| --- | --- |
| 三条纪律（不 sleep、不走捷径断言、证据不覆盖） | lab 的规格（地图 Notes 点名要原样带进 lab） |
| 12 处证据链接 + 四个坑的归因 | 被测仓的 `evidence/l2/README.md`，**在第一条场景切走之前就建** |
| 「两套装置」表、CI 只跑合成的两条理由 | 随 `scripts/l2/` 一起删（前者被 rig 声明取代，后者归 #13） |

## 8. 与相邻票的边界

- **#24（`remote-ops` 那 16 个脚本的归宿）**：本票只定「跨机场景骨架」（第 1 节那两条约束），
  `11/12/13` 的具体归宿仍归 #24。
- **#13（CI 接法）**：本票不被 #13 挡着——按第 7 节的顺序，第一条切的是真车场景，本来就不进 CI。
  逐条切给 #13 带来一个新约束：**迁移期间 CI 要同时面对「还在 L2 里的」与「已经在 lab 里的」
  两批场景。**

  **#13 于 2026-09-09 答完了那个约束**（[96-ci.md](96-ci.md) 第 5 节）：**不需要跨仓 checkout**
  ——任一时刻还没切走的完整住在被测仓、已切走的完整住在 lab，**没有哪一刻有一个 job 同时需要两个
  仓的场景代码**。

  但它补了本文第 7 节第 5 条守不住的那一半：**那条守的是「旧家不再跑」，「新家开始跑」没人守，
  而它是不出声的**（一条场景切进 `adapters/*/scenarios/` 却没被 `lab scenario list` 认出来，表现
  就是它安静地不存在）。#13 因此定了一步 `migration-drift`，按 #24 的「不腐烂」第 2 条把三份列表
  读回来比对，**读不回来也算失败**，保质期就是迁移期。
- **#25（证据的检索与保留）**：那 102 份在迁移完成时封顶，是一个有限且不再增长的集合。
