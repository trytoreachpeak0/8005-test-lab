<!-- 来源：#24 -->

# `remote-ops/` 与 lab 的分界：十六个脚本的归宿

`remote-ops/onboard-hmi/scripts/` 是今天**唯一在真车上跑过**的自动化。lab 上线之后，这十六个
脚本里哪些进 lab、哪些留下、留下的那半边靠什么不腐烂。

与 `25-l2-migration.md` 是姊妹篇：那份讲单机的 `scripts/l2/` 怎么跨机，这份讲已经跨机的这批
脚本怎么进模型。

## 1. 分堆的判据：调用方是谁

票的正文按「做什么」分三堆（通道建设 / 部署 / 待定）。**那个分法分不下十六个**——
`13-close-gates-when-idle.ps1` 与 `server-close-gates-watch.ps1` 三堆里一个都不属于。

分堆的判据是**这条路径的调用方是谁**：

- **调用方是 lab**（或迁移完成后会是）→ 进 lab，原件按 `25-l2-migration.md` 第 7 节切一条删一条；
- **调用方是人**，在 lab 不可用、不该用、或还不存在的时候 → 留在 `remote-ops/`，不删。

「它跨了几台机器」不是判据（传输层已经吸收掉了，见 `25-l2-migration.md` 第 1 节）；
「它是不是后路」也不是——**`11`/`12`/`13` 走的是 `05-setup-control-host.ps1` 配的同一份
ssh config、同一批 alias，而 lab 的传输层也是 ssh。lab 连不上车的时候它们同样连不上。**
真正的后路是那七个在车上跑、不需要 ssh 的（经 VNC 或控制台粘一行命令）。

## 2. 归宿表

| 脚本 | 行数 | 在哪台机器上跑 | 调用方 | 归宿 |
| --- | --- | --- | --- | --- |
| `01-probe` | 222 | 车 | 人 | 留 |
| `02-bootstrap-sshd` | 336 | 车 | 人 | 留 |
| `03-harden-sshd` | 114 | 车 | 人 | 留 |
| `04-serve-payload` | 227 | 控制端 | 人 | 留 |
| `05-setup-control-host` | 220 | 控制端 | 人 | 留 |
| `07-fix-clock` | 123 | 车 | 人 | 留 |
| `agvops-profile` | 14 | 车（装成 `$PROFILE`） | — | 留 |
| `06-deploy-onboard-hmi` | 501 | 控制端 | 人（CD） | 留，归 `40-deployment.md` |
| `08-deploy-slots-simulator` | 202 | 控制端 | 人（CD） | 留，归 `40-deployment.md` |
| `09-switch-io-module` | 135 | 控制端 | lab | `configure` 钩子 |
| `10-start-onboard-stack` | 277 | 控制端 | lab | 拆三份，见第 4 节 |
| `11-drive-journey` | 313 | 控制端 | lab | 一条场景 |
| `agv-drive-journey` | 290 | 车（交互桌面 session） | lab | `11` 的两个远端原语 |
| `12-reset-journey-state` | 296 | 控制端 | lab | 不是钩子，是一串钩子，见第 5 节 |
| `13-close-gates-when-idle` | 272 | 控制端 | lab | Watch 的控制端编排 + Repair |
| `server-close-gates-watch` | 324 | factory01 | lab | Probe |

共 3866 行（2026-09-09 数：`for f in *.ps1; do wc -l < $f; done`）。进 lab 的七个共 1907 行。

`01-probe.ps1` 的名字在 lab 的词汇下是错的（它做的是一次性只读读取，那个动作叫 read，见
`CONTEXT.md` 的 **Probe** 词条 `_Avoid_`）。**不改名**：词汇的管辖边界就是 lab 仓库的边界，
与 #17 定的「rig 名统一 kebab-case 只改 lab 侧」同一条。

## 3. 留下的那七个是 lab 的前置条件，不是 lab 的一部分

lab 的模型里没有「把一台机器带到可用状态」这一步——`10-registry-and-binding.md` 声明的是
**机器已经具备的能力**，而这七个脚本是**制造那些能力的过程**。

三条今天没有被表达、而 lab 整个建立在其上的依赖：

1. **`02-bootstrap-sshd.ps1` 造的账户结构。** 车上的生产账户 `HT` 是**空密码**，那正是机器
   开机直达桌面的原因；给它加密码会破坏 AGV 依赖的自动登录，放宽 `LimitBlankPasswordUse`
   会把它暴露在 SMB 上。所以 `02` 另建了 `agvops`。**lab 的桌面独占（计划任务进 session 1 +
   互斥体 `Global\W2G-InteractiveDesktop`）依赖的正是 `HT` 已经登录的那个 session，而 lab 里
   没有任何东西表达这个依赖。**
2. **`07-fix-clock.ps1` 做的时钟校正。** #9 定 lab 对时钟「run 起止各测一次、只记不校」。
   两条不冲突——`07` 是把机器带上线的一次动作，lab 的是运行期的每次 run。**但要补一条接缝**：
   lab 测出偏移之后报的 `reasonCode` 必须指向 `07`，否则一台没跑过 `07` 的车会让每条场景以
   一个查不出来源的失败告终。实测背景：agv01 与 agv02 首次接触时都快 158 天，且都指着一台
   错的 NTP（`172.19.206.222`，与真服务器 `172.19.205.222` 只差第三个八位组）；车载安全门
   拿服务端的 `observedAt` 与本机时钟比 5 秒窗口，钟偏到这个量级时任何一条干净会话都出不了
   `RecoveryRequired`。
3. **`agvops-profile.ps1` 的 UTF-8 强制。** 车上的中文 Windows 10 在 sshd 会话里起来是代码页
   936，pwsh 发 UTF-8，任何非 ASCII 输出到控制端就是乱码。**lab 从这些机器上读回来的每一个
   字节都经过它。**

**这三条写进接入文档的「已知限制」一节**（`20-adapter-contract.md` 的第 ⑥ 部分），不写进
registry——registry 声明能力，不声明能力是怎么来的。

## 4. `10-start-onboard-stack.ps1` 拆三份

正文问「被内核吸收还是作 `start` 钩子留在适配层」。**两个都不是完整答案，它拆三份**：

| `10` 里的东西 | 去哪 | 判据 |
| --- | --- | --- |
| **ORDER**：模拟器必须先在 1502 上听，客户端才能开 Modbus 连接 | 内核（角色的启动依赖） | 这是 Cast 内部的顺序，不是某一个角色的事 |
| **WAITING**：每一半确认起来了才走下一步（模拟器看端口、客户端看进程） | `ready` 钩子 | #4 已定 `ready` 必填且必须是「判据 + 超时」、禁止 sleep |
| 起停两个 WPF 应用本身 | `start` / `stop` 钩子 | #4 已定必填 |

起错顺序的后果不是「起不来」：客户端会在启动 IO 快照上失败、会话再也离不开
`RecoveryRequired`，**而那读起来像是服务端的问题**。这就是把 ORDER 放进内核而不是留给适配层
各自小心的理由。

**「刻意敲这条命令」那层授权不由脚本表达。** `10` 的 `.DESCRIPTION` 说授权不再由「你站在哪
栋楼里」表达、而由刻意调用它表达，没有任何东西会替你敲。lab 替人敲不是把它作废——
**授权在 lab 里由 Grant 表达**（`70-judgement.md`，#15），额度、次数、有效期、谁给的都在里面，
比「有没有一个脚本」严格。`in-service` rig 下这个钩子整个不存在（#23）。

**交互式 session 对驱动这一侧已经不承重。** `11-drive-journey.ps1` 的注释块写着：两个阶段现在
都是 loopback HTTP，「would run just as well over plain ssh in session 0」——UI Automation 那条
理由（只能看见自己 session 里的窗口）随 `8005-agv-onboard-hmi#14` 修好而消失了。**承重的是被测
的那两个 WPF 应用本身，不是驱动它们的脚本。** 角色的 `requiresInteractiveDesktop` 声明在被测
角色上，不在替身或驱动上。

## 5. `12-reset-journey-state.ps1` 不是一个钩子，是一串

正文问它是 `configure` 还是 `stage`。**两个都不是**：#4 已经定过同一个形状——「重装二进制不是
钩子，是 `stop → stage → configure → start`」。`12` 是

```
stop（两端）→ stage（两端各清各的）→ start
```

而且**钩子是按角色的**（`roles/<role>/`），不是按 rig 的。「rig 的 `stage` 钩子」这个说法本身
不成立：rig 声明的是角色组合，钩子挂在角色上。

**「两端一起清或都不清」的原子性，在这个分解下不需要新契约。** 2026-09-04 那次事故里，服务端
的旅程行被删、车上 `%LOCALAPPDATA%` 的 journal 还留着上一个 demand 的 revision 1，客户端按
契约报 `SNAPSHOT_REVISION_CONTENT_CONFLICT` 拆会话、重连、再撞，每七秒一次直到有人发现。

**那次的教训不是「清一端很危险」，是「被测系统自己出了声，而没有人在听」。** lab 里听它的地方
已经有了：

- `stage` 半途失败 → run 判 `ERROR`（#10 的三种结局之一），下一次 run 重走完整的一串；
- `stage` 全部报成功但两端不一致 → **`ready` 拦下来**，它本来就是「判据 + 超时」。

所以 **`ready` 的判据必须覆盖两端的一致性**，而不是「客户端进程起来了」。这与 #17 定的
「`stop` 钩子的 `completionCheck` 不能只判进程不在了」是同一个形状：**动作报成功不算数，
读回来才算数。**

**不扩 #4 的钩子契约，不加「部分完成」结局。**

两处在 lab 里会消失的东西：

- **停/起客户端那一段。** `12` 今天要停客户端是因为 journal 文件在它跑的时候是打开的；在钩子
  模型里 `stage` 本来就在 `start` 之前，客户端还没起来。
- **`-NoSimulator` 参数。** 它只因为「重启栈」这一步存在，而 `12` 的注释自己就写着「清那两个
  存储跟用哪个 IO 面毫无关系」。

**两端的备份改变归属。** 今天它们留在车上按运行次数堆积（2026-09-09 在 agv01 上实测 14 个
`journal-backup-*`，每个约 0.1 MB，#31 数的）。进 lab 之后它们是**证据**，进那次 run 的证据
目录（`50-capture-and-evidence.md`），由 run 的生命周期管。`12` 的注释块已经把理由写对了：
「they are the evidence for whatever made the reset necessary」——只是今天放错了地方。
这销掉 `45-footprint.md` 里 agv01 现场清单的那一格。

## 6. `09-switch-io-module.ps1` 是 `configure` 钩子，加一条顺序约束

**这一条不是本文新定的**：#15 第 11 节已经整段处理过这个脚本，并已按 `configure` 钩子对待它。
那节定下的两条，本文只是记在这里：

- **切 IO 指向的 `sideEffect` 是 `physical`**，按「它直接写入的东西所直接决定的行为」判，
  不做传递闭包。改写本身什么都不动（客户端启动时才读配置），但它让下一次启动变成
  「这台车会去驱动真实的锁和光幕」。
- **`configure` 的还原侧沿用 #9**：整份渲染 + run 后还原 + production tier 记 sha256 三连，
  还原失败判 run 失败。**Watch 不做配置改写**（#20），所以这一条只对 Run 生效。

正文的判断（rig 选择的一部分，不是场景能调的动词——否则一条场景可以在运行中把自己换到另一套
装置上）成立。补三条正文与 #15 都没有的：

1. **它必须在部署之后跑。** `09` 改的是车上已部署的 `appsettings.json` 的两个字段，而一次
   `06-deploy-onboard-hmi.ps1` 会重新渲染站点文件、把真模块地址写回去。脚本自己的注释写着这件
   事。在钩子序列里这是 `stage → configure` 的天然顺序，**但要求 `configure` 每次 run 都跑，
   不能因为「上次已经切过了」跳过。**
2. **`in-service` rig 下它不存在**（#23）：那个 rig 下 lab 不启动车载端，而 `09` 的前置条件正是
   「客户端没在跑」（它自己 `throw` 拒绝在运行时改 IO 目标）。
3. **`-NoSimulator` 那条防呆判据不进 lab**（#23 已定）：它防的是「`-NoSimulator` 打在一台指着
   loopback 的车上」，那是给人用的；rig 声明里两个角色在不在是静态的。

## 7. `11-drive-journey.ps1` 是一条场景

不是一组具名动词。理由**不是**「它跨了两台机器」——那个理由已经被传输层吸收掉了
（`25-l2-migration.md` 第 1 节）。理由是它的三步之间有两个承载正确性的等待。

### 7.1 三步的真实拓扑

脚本的注释块把三步标成 `vehicle / HERE / vehicle`，**其中 "HERE" 是「由控制端编排」的意思，
不是「在控制端执行」**：

| 步 | 做什么 | 在哪台机器上执行 | 在 lab 里是什么 |
| --- | --- | --- | --- |
| 1 | 提交子批次 | **agv01** — 客户端自动化面绑 `127.0.0.1` | 远端写原语 |
| 2 | 等 `WAITING_OPERATOR`，读出开了哪个仓位 | **factory01** — 读服务端 SQLite 的 `ProtocolInbox` | **远端读原语** |
| 3 | 放货进那个仓位、关门 | **agv01** — 模拟器自动化面绑 `127.0.0.1:58006` | 远端写原语 |

step 2 落在 factory01 上：`Invoke-Remote $ServerAlias`（`11-drive-journey.ps1:181,261`），
而它用的 `Microsoft.Data.Sqlite.dll` 是从服务器安装根里 `Get-ChildItem -Recurse` 找出来的——
**控制端上既没有那个数据库文件，也没有那个 dll。**

**所以切分点有两条判据，不是一条：**

1. **控制面绑在哪个地址上**（#17 定，覆盖自动化面那一类）；
2. **那台机器上才有的东西**——数据文件、依赖的程序集、只在本地可读的进程状态。

第二条是本节新增的。它解释了为什么 step 2 明明是「读一个事实」却不能在控制端做。

### 7.2 两个等待都在场景正文里

- **step 1 → step 2 之间**：等 `WAITING_OPERATOR`。这不是记账。2026-09-07 那次，整条 IO 轨迹
  落在 0.4 秒内、服务端判 `LOAD_RESULT_REQUIRES_RECOVERY`；前一次手工驱动的有 23 秒。
  `WAITING_OPERATOR` 是客户端在说它真的观察到门开了，别的信号都不是这个意思。
- **仓位循环**：循环是场景的（#10 定，次数由被测系统运行时决定），**而循环体里那个等待也是
  场景的**（#17 骨架第 3 条）。

两个都必须是场景正文里显式的 `Wait-LabCondition`，不能藏进「演完操作员」那个动词内部——
藏进去之后，别的场景复用那个动词会静默地继承或丢掉这个等待。

### 7.3 `agv-drive-journey.ps1` 是两个远端原语

它是 `11` 的车上那一半，由 `11` 部署并启动（#6 定的「远端半边每次 run 预置到 `workRoot`」的
真车实例）。它自己的注释已经写下了这条纪律：

> **Driving only, never asserting.** The one thing read back from the UI is whether input is
> accepted yet, which is a precondition for typing rather than a business fact.

**这条不是本份规格新定的，是这批脚本已经在守的。** 它正是 #17 骨架第 3 条的另一面：断言留在
场景正文里，远端原语只驱动。

### 7.4 `-ScanOnly` 的分界进 rig，不进参数

`-ScanOnly` 表达的是「step 1 不需要模拟器，step 2 和 3 需要」：真 Modbus 模块下门是真的电磁锁、
货是真的货，那两步归站在车边的人。**在 lab 里这不是一个开关，是 rig 的差别**——`in-service`
下那两步所在的场景整个不加载（#23）。

## 8. `13` 那一对：一个新词都不用造

`13-close-gates-when-idle.ps1` + `server-close-gates-watch.ps1` 是全工作区第一个「持续观察 +
自动动手」的东西，票的正文里没有它们（它们比票晚了不到两小时进仓）。它们逐条落在已有词条上：

| 它做什么 | 已有的词 |
| --- | --- |
| 盯着引擎往 SQLite 的写入 | **Watch** |
| 那一半在 factory01 上跑，理由是 2 秒窗口比一次 ssh 往返还短 | **Probe**（词条：「有些反应窗口比一次到控制端的往返还短，那种窗口只能在机器上赢」） |
| 每 250 ms 轮询同一个 SQLite 文件 | **Sample ring** |
| 旅程 `Completed` 就地 `Stop-Service` | **Repair**（起停是修复的手段，不是观察的手段） |
| 旅程 `Blocked` 时**不停**服务，报告了事 | **Repair** 与 **Precondition** 两个词条里那个先例 |
| 停完读回两个 bool 与服务状态确认真的关上了 | **Repair**（「结果与验证分开记」） |

**Repair 与 Precondition 词条里那句「一次真车上的先例里，旅程走到终态 Blocked 时正确的动作
恰恰是不停服务，因为停了就把恢复所需的那个循环拿走了」——那就是这个脚本。** #22 与 #15 已经
把它当先例引进词汇表，只是没有人写下「所以它的归宿就是 Watch + Probe + Repair」。

### 8.1 Probe 的作用域扩到 run

`13` 同时被用在两个场合：跑一次真车场景时开着（`11` 的注释：「Run 13-close-gates-when-idle.ps1
alongside a real trip」），以及生产上长开着防止任何一次意外的旅程完成后接下一单。

**Probe 词条今天只定义了 watch 作用域**（「watch 开始时部署、结束时清除，从不常驻」）。
**定：Probe 的作用域扩到 run，语义不变——run 开始时部署、结束时清除，从不常驻。**

理由不是 `13` 一个用例。是 Probe 存在的那条理由——「有些反应窗口比一次到控制端的往返还短」——
**与 Run/Watch 之分无关，它是拓扑的性质，而 Run 一样跨机**。#20 定的「按延迟切而非按功能切」
在两个作用域下是同一条。

**这条的直接后果：`13` 的 Probe 与 `11` 的车上半边是 `real-onboard-normal-load` 这条场景在两台
机器上的两半，它们本来就一起切。** 不存在「Watch 那半边要不要跟着第一条场景切」这个问题——
#17 那五条判据的第 2 条（「角色分布在两台以上机器上跑完，不接受单机过渡态」）已经把它定了。

### 8.2 判据只声明会让 lab 改变动作的取值

`13` 的判定有三条穷尽的分支：`Completed` → 停；`Blocked` → 报告，不停；其余 → 继续等，
超时报告。**`BlockReasonCode` 只进心跳，不参与判定。**

**这是对的，必须原样带进 lab**，理由是被测系统那一侧的取值集合是**开放的**。
2026-09-09 实测 `8005-agv-control-server` 里 `BlockReasonCode` 的赋值（筛法：
`grep -rhE 'BlockReasonCode *=' src/ --include=*.cs --exclude-dir=Migrations`，24 处；
用一个不存在的字段名校准得 0）：

| 形态 | 处数 | 静态可读出取值？ |
| --- | --- | --- |
| 置 `null`（解除阻塞） | 6 | — |
| 裸字面量 | 6（**5 个唯一值**） | 是 |
| 具名常量 | 1（`StationTimeoutDoorNotClosedReason`） | 是，但要跟到常量定义处 |
| 运行时构造 | 5（`$"{legName}_{result.Outcome}"`、`$"CHARGER_{dispatch.Outcome}"`、`Outcome.ToString()`、两处条件表达式） | **否** |
| 转发别处的变量 | 6（`rejection.ReasonCode`、`terminalReasonCode`、`notEngaged`…） | **否** |

**24 处里只有 7 处能静态读出取值，11 处要到运行时才知道。任何一份写死的取值集合都会立刻
产生假阳性。**

所以：**判据只声明「哪些取值会让 lab 做出不同的动作」，其余一律落进 catch-all 并进账。**
这正是 **Sample ring** 词条那句「采得比判得多是对的，但采而不判的部分绝不参与判定，它只进账」
在一个真实被测系统上的样子。

**不做 schema 漂移检查。** 表名与列名的漂移由 SQL 自己炸（SQLite 查不存在的列会报错）；
语义漂移由 catch-all 兜住并进账。一个例子：2026-09-09 11:00 控制端加了一个新状态
（`stage` 仍是 `AwaitingSublot`、`BlockReasonCode = STATION_TIMEOUT_DOOR_NOT_CLOSED`，
表示车看得见地等着人关仓门）。`13` 今天遇到它会正确地继续等，并把那个码打进心跳——
**这不是缺陷，这是设计对了。**

### 8.3 一格缺口，不归本文

「车在等人关门」这件事 `13` 记下了，但**不会送出去**。触发条件的表达归 `70-judgement.md`
（不变量），送达归 `65-courier.md`（Courier）。本文只指出这条不变量有主了。

### 8.4 `13` 的 Repair 今天没有过安全门

它只有一个 `-Force`（降 `$ConfirmPreference`，因为 `ConfirmImpact = High` 在无 host UI 时会
`throw` 而不是提示）。进 lab 之后 `Stop-Service` 是一次 Repair，必须过 `70-judgement.md` 那道门。
判定结果与 Grant 的形态归 #15，不在本文。

## 9. 留下的那半边靠什么不腐烂

答案不是纪律，是**让副本消失**；消失不了的，**让腐烂出声**。

### 9.1 今天已经在腐烂的六处（2026-09-09 实测）

| 副本 | 形态 | 已知状态 |
| --- | --- | --- |
| `06-deploy-onboard-hmi.ps1` 里的配置校验 | 逐条复刻 `SQCD.Agv.Infrastructure/Configuration.cs` 的启动校验 | #12 已定抽成共享判定、`06` 反过来引用 |
| 4 个脚本里的 15 处 SQL、7 张被测系统的表 | `JourneyRuntimes` `OrderIntents` `StationOperations` `AcceptedDemands` `SessionRecoveries` `ProtocolInbox` `ProtocolOutbox` | 见 9.2 |
| `08-deploy-slots-simulator.ps1` 的 `$SdkVersion` | 写死 `8.0.425`，跟随 `8005-agv-program` 的工具链基线 | 2026-09-09 10:31 手工同步过一次 |
| `09` 改的两个字段 | 一次 `06` 会把它写回去 | 脚本自己的注释写着，无机制 |
| 桌面快捷方式与安装根 | `8005 AGV 车载端.lnk` 是 09-07 20:34，安装根 `C:\8005\OnboardHmi` 是 09-08 10:06 | **最后一次换安装根没走 `06`**（#31 实测） |
| `11`/`13` 判据里的旅程状态语义 | `Completed` / `Blocked` / 其余 | 2026-09-09 11:00 被测系统加了一个新的中间态，无通知 |

### 9.2 三条规则

1. **凡是能变成「一个地方 + 两个调用点」的，就那么做。** #16 已定形式：纯函数模块放在
   `adapters/<name>/shared/`，入 JSON 出 JSON，不认 `$Context`、不认 `runId`、不发起任何传输；
   `remote-ops/` 的脚本反过来引用它，找不到就**硬失败并提示跑 `setup.ps1`**，不静默回退到
   自己那份旧副本。
   **它解不了 SQL 那一类**——SQL 不是判定，是读取，而读取要经传输层。那 15 处 SQL 随
   `11`/`12`/`13` 一起进 lab，落在适配层的 `db` 控制面声明里（#4 定的内核 kind），
   **表名与列名只出现在那一处**，场景与动词只认具名查询。留在 `remote-ops/` 的脚本里不再有 SQL。
2. **消失不了的，加一条把两份读回来比对的守卫；守卫加不上的，标注保质期。**

   **先例，2026-09-09 12:44 落地于 `8005-agv-control-server` 的 `59a3c38`**，形状与本节要解的
   问题完全一致：G3 runner 里协议身份有两份，PowerShell 侧一份、嵌入的 C# 合成对端里一份，
   而后者活在 `@'...'@` 里——**那种 here-string 不插值，所以它不可能引用前者，只能是第二份
   手抄**。改一份不够，跑起来立刻 `INCONCLUSIVE_RUNNER_ERROR`。

   那次的解法是三条，本文原样采用：

   - **here-string 结束后用正则把六个值读回来与另一份比对，不一致就 `throw` 并指名道姓
     是哪一项、两边各是什么**；
   - **价值在于失败得早**——原来那条错误要等四个仓克隆完才抛，新守卫在任何克隆之前触发；
   - **读不回来也算失败**：模式失配意味着守卫瞎了，**该修模式而不是删掉它**。

   最后一条是三条里最重的，因为它是「静默通过」这个失败模式的正面解法——一个读不到东西的
   检查报 CLEAN，比没有检查更糟。

   **本节列的六处里至少两处直接适用**：`08` 的 `$SdkVersion`（写死了一个跟工具链基线走的值,
   两处：它自己与 `factory-server/scripts/07-install-dotnet-sdk-guest.ps1`）、`09` 与 `06` 之间
   那个「改完会被写回去」的关系。

   **守卫加不上的**（比如「这条路径最后一次在真车上跑通是什么时候」——跑一次真车太贵，
   没法做成检查），退到标注：脚本头上一行，最后一次在真车上跑通的日期加那次被测系统的 commit。
   这不是检查，是一句**让读的人知道自己在信一份多旧的东西**的话。
3. **一次依赖的上限，一次定清。** `remote-ops/` 依赖 `repos/8005-test-lab/` 存在，而工作区根
   `.gitignore` 排除了 `/repos/`（由 `setup.ps1` 恢复），所以一台刚装好、还没跑过 `setup.ps1`
   的控制端用不了带这种依赖的脚本。

   **上限两条**：①**只有共享判定这一种依赖**（纯函数模块，入 JSON 出 JSON）；
   ②**只有部署那一类脚本可以有**。今天符合的只有 `06`（它里面有两处复刻的校验）——
   **`08` 今天一处都没有**（2026-09-09 实测），它是这条上限覆盖但尚未用到的那一格。

   留下的其余七个（`01`–`05`、`07`、`agvops-profile`）**一个都不许依赖 lab**——它们是 lab 还
   不存在时把机器带上线的那条路径，给它们加 lab 依赖等于把后路接到前路上。

## 10. `Get-WireToGateStatus.ps1`：留下，同时在 lab 里有一个孪生

`remote-ops/status/Get-WireToGateStatus.ps1`（519 行）在控制端上跑，纯读，五个信号源，每天
一个 JSON Lines 文件。它是今天「解票期间确认车 idle」的事实标准入口。

- **留在 `remote-ops/`**：它的调用方是人——agent 在解票时、现场排查时，而那时候 lab 可能整个
  还没装。
- **同时在 lab 里有一个孪生的只读诊断场景**（#22 定的 Diagnostic scenario）。**这不是双跑**：
  两者的调用方不同（人 vs watch 自动拉起）、触发条件不同、产物不同（本地 JSONL vs 一次 run 的
  Evidence）。#17 那条「不双跑」针对的是同一个调用方的同一条路径有两份实现。
- **孪生那一份不自己维护 JSONL。** 那份日志在 lab 里由 Observation 承载。脚本自己的注释已经
  把这件事说对了：「This file carries nothing the product depends on: deleting it loses history
  and breaks nothing.」

## 11. 与相邻票的边界

- **`40-deployment.md`（#12）**：`06` 与 `08` 归它。本文只记下 `08` 于 2026-09-09 10:31 跟随
  SDK 基线改过 `$SdkVersion`，以及它作为「写死了一个别处会变的值」的样本。
- **`70-judgement.md`（#15）**：`10` 的授权、`13` 的 `Stop-Service` 过哪道门，归它。
  本文只定归宿，不定安全门。
- **`65-courier.md`（#28）+ `70-judgement.md`（#21）**：8.3 那一格。
- **`25-l2-migration.md`（#17）**：`11` 与 `13` 的 Probe 是第一条跨机场景的两台机器，
  一起切。第 7.1 节更正了那份规格里的一处拓扑描述。
- **#13（CI 接法）**：本文不碰 CI。留下的七个脚本一个都不进 CI——它们的调用方是人。
