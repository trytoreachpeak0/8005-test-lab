<!-- 来源：#31 -->

# 足迹：lab 在机器上留下什么，谁清，什么时候清

## 1. 两维分类

足迹按两维分。**这两维不是描述，是判定输入**——第一维决定谁能清它，第二维决定要不要淘汰策略。

**第一维：这是谁的东西。**

| 值 | 含义 | `sideEffect` |
| --- | --- | --- |
| `lab` | 只有 lab 用，删了只影响 lab | `none` |
| `sut` | lab 放的，但被测系统在用，删了被测系统会变 | `process`，且 `reversible: false` |

`sut` 那一档因此落进 #15 的「不可逆的 `process` 按 `physical` 判」，在 `tier: production` 上需要
当场发起、TTL ≤ 15 分钟不可覆盖的凭证。**不另写一条 tier 判断**——那是第二道门，这张图不加第二道门。

**第二维：它会不会自己消失。**

| | 自己会消失 | 被覆盖 / 不增长 | 会积累 |
| --- | --- | --- | --- |
| **`lab`** | 探针（#20 硬上限自杀）<br>桌面互斥体 `Global\W2G-InteractiveDesktop`<br>`<workRoot>/selftest/<runId>`（#29）<br>回捞临时件（第 6 节） | 远端半边 `Lab.Remote` + `tests/unit` + `tests/machine`（内容哈希同步）<br>Vigil 心跳 JSON（覆盖写）<br>Vigil 计划任务与 `vigil/Watch-LabVigil.ps1` | 构建缓存（第 4 节）<br>stage 目录（第 5 节）<br>Toolkit（按 sha256，升级即并存）<br>`evidence/` `observations/`（归 #25 #26）<br>Vigil 的 courier 配置副本（第 7 节） |
| **`sut`** | — | 安装根 + 一代 `.previous`<br>桌面快捷方式<br>机器级凭据环境变量（第 8 节） | — |

**只有右下角那一格需要淘汰策略；只有下面那一行需要过安全门。**

## 2. 两条例外

**`HALT` 哨兵永远不在清理范围内。** #15 把它定在探针 `workRoot` 根，存在即降只读，恢复只能靠
`lab watch grant`（删文件不恢复）。它在 `workRoot` 里但不属于上表任何一格——它是人放进来的，
lab 只读它。清理动作扫 `workRoot` 时具名跳过它，且跳过是一条带理由的项，不是「不在列表里」。

**`evidence/` 与 `observations/` 是产物不是残留。** `lab machine sweep` 绝不碰它们。
保留与检索归 #25 与 #26。

## 3. 清单怎么来：声明 + 实测，两边对账

**不能有一份集中的手写清单**（`8005-mes-ingest/Get-WpfDesktopTestFilter.ps1` 的注释记着那个先例：
第一版手写排掉两个类名，下一次跑就有两个新类失败）。

- **声明贴着产生足迹的那段代码**，形状沿用 #29 的 `$LabTestFootprint.Touches`
  （`Processes` / `Ports` / `Desktop` / `Paths`），**从 `tests/machine/` 扩到适配层钩子与动词**。
  没有声明 → 加载期拒绝，`footprintDeclarationMissing`。
- **`lab machine footprint` 做的是对账，不是列表**。声明的那份 × 从机器上实测读回来的那份，出三格：

  | 格 | 含义 |
  | --- | --- |
  | 声明了、机器上没有 | 已清或从未创建 |
  | 声明了、机器上有 | 正常 |
  | **机器上有、没人声明** | **`footprintUndeclared`** —— 这一格是这条命令唯一真正值得跑的理由 |

- 它是纯读，`sideEffect: none`，生产车上无条件允许，与 `lab machine check` 同级。
  够不着的机器记 `notAttempted`（#29 定的值，不判红也绝不算绿）。

**「机器本来就有的洞」与「lab 留下的东西」分开记。** `lab machine footprint` 只认自己声明过的东西，
其余一律报成「机器上还有这些，来源未知」，绝不报成 lab 的足迹。`remote-ops/` 留下的东西归 #24。

## 4. 构建缓存

键沿用 #12 的 `<角色>-<commit>-<flavor>`，落构建机的 `workRoot`。**构建机缺省 `local`，
三台车都不是构建机，所以构建缓存从不落在生产机器上。**

| flavor | 单份实测（2026-09-09） |
| --- | --- |
| framework-dependent | 车载端 6.6 MB、模拟器 1.9 MB |
| self-contained | 车载端 186.0 MB、模拟器 184.8 MB |

- **上限按 flavor 分，保留最近 N 个 commit**：`framework-dependent` 缺省 20，`self-contained`
  缺省 10。两个数可在 `registry.json` 的「构建机」**能力**上覆盖——覆盖的是能力的一个参数，
  不是「这台机器的清理策略」（#5：登记册只声明能力）。
- **不用 LRU，不用任何 NTFS 时间戳。** `fsutil behavior query DisableLastAccess` 实测三台机器
  同值（`2`，System Managed）不同行为：控制端与 `vm01` 是 ENABLED，`agv01` 是 DISABLED。
  在 `agv01` 上那些 `LastAccessTime` **不是一律等于 mtime**（17 个条目里 15 个相等、2 个不等），
  它们是**任意的**——有的等于 mtime，有的是某次 `Move-Item` 留下的，**没有一个的含义是「最近被
  用过」**，而这件事在带外看不出来。改成 lab 自己记：**#12 已定的 `publish.ok` 戳文件里加
  `lastUsedAt`**，取缓存时刷新，不新增文件。
- **淘汰跑在 `publish` 钩子取缓存的那一刻**，不另起时机——一个额外的触发器就是又一个会静默不跑
  的东西。
- **撞上限先淘汰再继续，不拒绝。** 与 #15 的 fail-closed 有意不同向：那条是「判不出被测系统的
  状态就别动它」，这里判得很清楚，拒绝一次 run 换不来安全。

**Toolkit 不自动淘汰。** #16 定了升级即并存、回退只是改一行清单，自动淘汰会把回退的路删掉；
一份 ≈ 45 MB，一年可能一份都不长。`lab toolkit verify` 报出「这台机器上有几份、`toolkit.json`
只认哪一份」，多余的由 `lab machine sweep --toolkit` 显式清。

## 5. stage 目录：无条件清

**不延续 L2 那条「PASS 就删、失败才留」。** 实测（2026-09-09，控制端）：那条规则落地后的 77 次
PASS，77 个目录一个没删掉——`Remove-Item -Recurse -Force -ErrorAction SilentlyContinue` 每次都
部分成功（publish 副本删了，SQLite 的 `-wal`/`-shm` 被句柄占着删不掉），静默失败 77 次。
六天累计 199 个目录、254 MB。**PASS 与 FAIL 的目录内容完全相同，两个分支塌成了同一个行为。**

而那 199 个其实是**三类**：

| 类 | 特征 | 数 |
| --- | --- | --- |
| PASS，删除部分成功 | 只剩 SQLite 三件套 | 77 |
| FAIL，走 else 分支不删 | SQLite 三件套（synthetic rig 本来就没有子目录） | 21 |
| **编排在 `finally` 之前炸了** | 连一份证据都没有，一部分还带着完整的对端 publish 副本 | **98** |

**第三类比前两类加起来还多**，而 #21 与 #29 早就点过「编排自己炸了」不是例外是一个持续产生的
类别。这一类跟「留给人诊断」这个理由完全无关——没有证据就没有 runId 的上下文。

- **stage 目录无条件清，PASS 与 FAIL 一视同仁**，带 `completionCheck`（第 9 节）。
- **那条规则想保住的东西改由 #11 保住**：`controlserver.db` 这类「唯一写着原因的地方」按 #11 已定的
  做法进 Capture（db 导表内容 JSON 加库文件 sha256 锚点，不整库拷回），落在一个会被封存、有
  `MANIFEST.sha256` 与 `completeness.json` 的地方，而不是靠一次删除失败活着。

## 6. 回捞的临时件

落 `<workRoot>/capture/<runId|分片 id>/`，**传成功即删**。

**例外：传失败时不删，并在 `completeness.json` 里记下它在目标机上的路径。** #11 定了回捞失败不改
run 结局、另立先列清单再核销的账；加上这一条之后那份账能分开三件事——**从未尝试** /
**捞了但没传回来，东西还在那台机器上，路径在这里** / **捞回来了**。中间那一格是唯一还能补救的一格。

## 7. `vm01` 上的常驻物（#30）

心跳 JSON 覆盖写不增长；Vigil 计划任务与 `Watch-LabVigil.ps1` 故意留下，不在自动清理范围。

**courier 配置副本是凭据，而 `vm01` 上 ACL 挡不住第五个 runner。** 实测：四个
`actions.runner.*` 服务以 `NT AUTHORITY\NETWORK SERVICE` 跑，第五个
（`\GitHubRunner-win11-01-mes-ingest-desktop` 计划任务）以 `agvops` + `RunLevel=Highest` 跑，
而 `agvops` 是本机 Administrators 成员。

- **ACL 给 SYSTEM + Administrators**，去掉 Users 与 Authenticated Users 的继承。这挡住四个
  `NETWORK SERVICE` 的 runner。
- **明说它挡不住那个提权的 desktop runner**，写进接入文档——照 #22 对 ledger 的做法，
  不假装能挡。**#13 把这句从一个变成两个**：lab 自己的第二个 runner
  （`win11-01-test-lab-desktop`）也必须以 `agvops` + `RunLevel=Highest` 跑，否则它拿不到桌面
  互斥体（[96-ci.md](96-ci.md) 第 4.4 节实测了默认 DACL）。**挡不住的那一个里，从此有一个是
  lab 自己。**
- **不做 DPAPI 机器级加密**：对本机管理员透明，是假的防护。
- **Vigil 用一份与控制端 Courier 不同的 webhook key**，理由不是更安全（同一个群，泄漏后果一样），
  而是**可撤销性**——泄漏时能单独换掉它。
- **位置不在 `C:\actions-runner` 之下，也不在任何 runner 的 `_work` 之下。**

**Vigil 的停用**：它分不清「lab 不用了」和「控制端挂了」，所以不自己决定停（#30：假装能区分就是
又一扇开着的门）。改成让控制端在还能说话的时候留下一句话——

- `lab watch stop` 停掉**最后一段** watch 时，往心跳文件写 `intentionalEnd`；Vigil 读到就不再报
  「听不到」，但每 UTC 日那一条照发，内容变成「lab 已有序收工，我还在这里」。
- 异常终止时那句话不会被写下，Vigil 继续报「我听不到控制端」——那时它确实该报。
- 日报带上「我在等的那几段 watch 已经 N 天没出现过」，停用的决定推给读日报的人。

## 8. 机器级环境变量：#12 与 #33 的冲突，按 tier 裁决

- **lab 不为自己在任何机器上写环境变量**（#33 那句的这一半完全成立；`WINAPP_CLI_TELEMETRY_OPTOUT`
  设进程环境不设机器级，#16 已定）。
- **适配层的 `stage` 钩子写的是被测系统启动所需的配置**，那是 `sut` 那一档，不是 lab 的配置。
  一并禁掉等于 lab 装出来的东西启动不了。
- **按 tier 分**：`tier: production` 上**不写**——那两个变量已经由 CD 放在车上，lab 重写只是拿一个
  可能过时的值覆盖一个正确的值；lab **只读回核对**。`tier: lab` 上**写**——那些机器上的安装本来
  就是 lab 放的。
- **桌面快捷方式同一条**：production 上不建（06 已建，路径固定不会过期），lab tier 上建。

两者都在第 1 节表的 `sut` 那一行，都不在 `lab machine sweep` 的范围内。

## 9. 清理：一个必须核销的动作

**`completionCheck` 的必填判据不是「危不危险」，是「做没做成会不会不出声」。** 这收窄了 #22 原定的
必填范围（原本与 #15 那两个额度重合，即 `physical` 加不可逆 `process`）。

**凡是删除、凡是「让某样东西不存在」的动作，无论 `sideEffect` 是什么，必须有 `completionCheck`**
——「删了」在 Windows 上的失败模式就是静默的。结果按 #22 已定的做法记：**`verification` 与
`outcome` 分开记**。

- **触发**：每次 run / watch 收尾**自动清 lab 自己的东西**，且核销。核销失败**不改 run 结局**
  （#11：传输的失败与被测系统的判定是两类事），落 `sweep` 结果 JSON 并计入 #28 第 2 类事件
  （lab 自己坏了那一档）。
- **`sweep` 的结果形状**照 `RELEASE-CANDIDATE.md` 第 9 节的卸载结果 JSON：逐项列**打算删什么、
  删完再读回来还在不在**，而不是「发了几条删除命令」。
- **子命令**：`lab machine check` / `footprint` / `sweep`，并进 `lab machine` 那一行，
  **子命令表仍是十二行**。`sweep` 的开关：`--dry-run`（**缺省，只报不删**，依据是 RC 第 9 节
  「`-ConfirmUninstall` 是必需的显式授权」——这个项目对「删」的既有严格性等级就是默认不删）、
  `--toolkit`、`--all`（一台机器退出 lab 时用）。
- **删被测系统的东西不是内核能力**，是适配层动词，走 `lab do`，因此自动落进 #15 的门与账。

## 10. 计划任务：让它自己消失，不做扫除

「孤儿」这个概念对大多数东西不成立——探针有硬上限自杀（#20）、`selftest/<runId>` 跑完即清（#29）、
桌面互斥体随进程退出。**唯一真正危险的孤儿是桌面计划任务**：`10-start-onboard-stack.ps1:162-169`
在 `finally` 里 `Unregister`，注释说「留下它会把下一次登录变成一次未授权的启动」，
而 #30 已经把「控制端会死」定成必须假设的事实。

- **命名** `Lab-<用途>-<runId 后 8 位>`，这样它在 `footprint` 对账里能被认领。
- **触发器对象上设 `EndBoundary`，设置集上给 `-DeleteExpiredTaskAfter`**，任务的寿命上限等于它
  要启动的那个动作的超时上限。控制端正常活着时 `finally` 里的 `Unregister` 照旧——**自毁是兜底
  不是替代**。
- **一条实测欠账**：`-DeleteExpiredTaskAfter` 参数存在（控制端 `Get-Command` 核过，类型
  `TimeSpan`），但「一个从未触发过的任务到了 `EndBoundary` 会不会真的被删」没有实测——那要在机器上
  真建一个任务，是写操作。按第 9 节自己那条判据，**它坏掉时是静的**，所以实现时必须带核销
  （下一次连上这台机器时对账那个任务名还在不在）。落点与 #36 同类。

## 11. `reasonCode`

三个，进 `src/Lab.Judge/reason-codes.json`（放 `Lab.Judge` 因为 `sweep` 的核销发生在目标机上，
`Lab.Remote` 与 `Lab.Probe` 两处都要发得出来）：

| 码 | 何时 | `retryDisposition` |
| --- | --- | --- |
| `footprintDeclarationMissing` | 加载期，一段会在机器上留东西的代码没有声明 | 不可重试 |
| `footprintUndeclared` | 对账时，机器上有 lab 特征的东西但不在任何声明里 | 不可重试 |
| `sweepIncomplete` | 清了，核销发现没清掉 | 可重试 |
