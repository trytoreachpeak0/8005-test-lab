<!-- 来源：#30 -->

# Vigil：谁看着控制端自己

Courier（`65-courier.md`）是「发」的那一半，**Vigil 是「等」的那一半**。

一条只在异常时才出声的通路，无法证明自己还活着；一条无条件出声的通路，它的缺席本身就是信号——
**但前提是有东西在等它**。Vigil 就是那个等的东西，而它必须在被等者之外。

## 1. 它覆盖的三类死法

看着控制端的东西只看得见外面，所以按「从外面看得见什么」切，不按「什么坏了」切。

| 类 | 死法 | 外面看见什么 | 谁发现 |
| --- | --- | --- | --- |
| 1 | 进程没了、机器关了、网断了、笔记本被带走 | 存活信号停 | Vigil |
| 2 | 编排卡在一个不返回的调用上、Modern Standby 节流 | 取决于存活信号由谁发 | Vigil，靠第 2 节 |
| 3 | 活着且在干活，但送达通道断了 | 存活信号还在，事件送不出去 | Vigil，靠第 3 节的两个字段 |

控制端是一台**笔记本**（`LAB-WIN-01`，HP ProBook 445 G11，Wi-Fi，电池）。它的空闲待机超时配的是
「从不」，但它支持 S0 Modern Standby / Network Connected，而且**它会被人合盖拎起来走**。这不是
机房里的一台服务器，死法清单要按这个事实写。

## 2. 存活信号是「最近一次真的写下了什么」，不是一条定时心跳

**存活信号必须由做事的那条路发，不能由一个独立的定时器发。**独立定时器在编排卡死时照样跳，
那就是「心跳还在但没人干活」。

形状取自被测系统 `JourneyRuntimeEngine.cs:1553-1561`：服务端不看心跳，看 `lastInboundAt` 的年龄
（`MaximumEvidenceAge`，30 秒），而且**任何入站消息都算，不只是心跳**。

所以 `lastProgressAt` 由**采样环回传、`violations.jsonl` 追加、账追加、分片写入**——任何一件刷新。
**卡死就是不刷新**，不需要另外去检测「卡住」。

反面教材是同一个仓的
`docs/defects/20260908-session-recovery-required-never-clears-while-vehicle-idle.md`：车载端只在状态
变化时才发 `RecoveryStateReport`，journey 跑完车静止时那份报告永远不来，会话卡死 6 分 36 秒。
**最需要它的静止态，恰恰是它永远不来的那一种。**

## 3. 载荷

控制端推一行 JSON 到 `vm01`，覆盖写同一个文件：

| 字段 | 含义 |
| --- | --- |
| `lastProgressAt` | 第 2 节那个语义的 UTC 时刻 |
| `watchId` | 哪一段 watch |
| `boundProductionMachines` | 动着哪些 `tier: production` 的机器 |
| `courierPendingCount` | Courier 队列里 `pending` 的条数 |
| `courierOldestPendingAt` | 最早那条 `pending` 的时刻 |
| `grantExpiresAt` | 探针手上那份额度的到期时刻（#15 的两个钟取先到） |

**后两个 courier 字段覆盖第 1 节第 3 类死法，代价是零**：控制端活着、心跳照推，但送达断了，
`courierPendingCount` 会涨，而 Vigil 在另一条路的另一端。

**这是「发现问题的东西与报告问题的东西是同一条路」的解**：不给它配第二条报告路，而是让心跳
捎上报告路的健康度，判定发生在别处。

## 4. 通路：推，不是拉

```
控制端 LAB-WIN-01 --ssh 经 factory01 跳板--> vm01（Vigil）--直连--> IM 群
       |                                                              ^
       +------------------ Courier 走自己那条路 -----------------------+
```

`vm01` 上一个计划任务定期检查那个文件的年龄，过期就用**同一份 courier 配置、往同一个群**发。

**拉（`vm01` 经 SMB / WinRM 读控制端本机的心跳文件）在网络上成立但被排掉**，三条理由：

1. 拉要给 `vm01` 一份能读控制端文件系统的凭据，而推只用控制端**已经有的**那把
   `~/.ssh/agv_vm_ed25519`。`vm01` 上跑着五个 self-hosted runner。
2. 控制端是一台会换网的笔记本。拉要求 `vm01` 知道去哪里找它，换个 Wi-Fi 就变成**不会自愈的假阳性**。
3. 拉判「文件多久没变」，推判「有没有人来推过」——后者顺带证明了控制端主动做成了一件跨机器的事。

## 5. 它发的是同一个群，不是第二条通路

被测系统 `OnboardMessageProcessor.cs` 那条 OPEN 注释定的纪律：

> If that turns out not to happen, **widen this rather than adding a second announcement somewhere else.**

**Vigil 不是第二条通告路径，它是同一条通路多一个发起方。**同一份 `--courier` 配置文件（`65-courier.md`
第 1 节那五个字段一字不改）、同一个群、同一条「`successPath` 等于 `successValue` 才算收下」的判据。

判据是 body 不是 HTTP 状态码——这条源头是 `RELEASE-CANDIDATE.md` 第 5 节（「判定标准是读回的 body，
不是退出码为 0，更不是 ping 通」），#28 是第二次应用，本节是第三次。

## 6. 阈值

| 项 | 值 | 理由 |
| --- | --- | --- |
| 推送间隔 | 5 分钟 | |
| Vigil 检查间隔 | 5 分钟 | |
| `lastProgressAt` 年龄阈值 | **15 分钟** | 等于 #15 定的 production `physical` TTL 上限：能让车动的授权最多再活 15 分钟 |
| `courierPendingCount` 阈值 | 20 条**或**最早一条超 60 分钟 | 沿用 `65-courier.md` 第 12 节，同一件事不该有两套数 |
| Vigil 自己的存活播报 | 每 UTC 日一条 | 第 9 节 |

四个阈值**全部归内核**，是具名常量，改它要改代码过 review。

## 7. 偏向假阳性

跳板不通、网络抖动 → 心跳送不到 → Vigil 误报。**这是假阳性，代价是群里多一条假警报**；假阴性的
代价是探针带着额度继续修、账没人捞、人以为有车看着。**这个部件应当偏向假阳性**，与 #15 的
fail-closed 同向。

**Vigil 那条消息必须说「我听不到控制端」，不能写成「控制端挂了」**——它区分不了这两件事。

报警正文必须带上 `boundProductionMachines`，并写明「这些机器上可能有攒着未回传的账，下一次
`lab watch start` 会先回捞」——这条消息的读者正处在唯一能补救的位置上，而补救动作恰好是他本来就
要做的那一个。

## 8. Vigil 不加载 lab 的任何模块

`90-repository-layout.md` 那四个内核模块的分界是「在哪台机器上加载」。**Vigil 在第五个位置上，
而它不进那张表**：它是 `vigil/Watch-LabVigil.ps1` 一个**自足的单文件 pwsh 脚本**，不
`Import-Module` lab 的任何东西，不读 `registry.json`，只读推上来的那行 JSON 与那份 courier 配置。

理由是同一条判据：**如果 Vigil 依赖 lab 的模块，lab 仓库的一次坏改动会同时干掉 watch 和看着 watch
的那个东西**，而那次失败不出声。

代价是它要自己实现 webhook POST 与 body 判定，与 `Lab.Core` 里那份重复约几十行。**这份重复是故意的**：
`90-repository-layout.md` 那条「凡是要被 lab 之外调用的适配层判定必须是纯函数模块」要消除分叉，
本条要消除共同故障，两者方向相反而考虑同源。

## 9. v1 的止损线

**只做一层，而且明说。**「监控者的监控」的递归不终止于做够几层，终止于**最后一层发给一个 lab
管不着的接收者**——群里的人。

1. **Vigil 自己也无条件发**：每 UTC 日一条「我还在，本日看着 N 段 watch」。这是同一条判据的第二次
   应用，不是又加一层。
2. **接入文档写死三句**，与 `65-courier.md` 第 7 节那句「通道已收 ≠ 人知道了」并列：
   - **watch 不是无人值守的**（`RELEASE-CANDIDATE.md` 第 11 节的直接后果——真车动作要逐次授权加
     现场物理安全 GO，每次都要、不可复用；RC 第 12 节承认的无人值守边界停在「合成对端、无移动」）。
   - **没有东西在等 Vigil。**`vm01` 与控制端同时不可用时整套东西静默失效，没有任何迹象。
   - **没收到摘要不代表一切正常**（`65-courier.md` 第 4 节空摘要不发，那条不改）；
     **没收到 Vigil 的日报**才是「有东西不对而且没人知道」的信号。

## 10. 落点

- 心跳写出在 **`Lab.Core`**（只在控制端加载），**探针侧零新增**。
- `--vigil <文件>`：凭据与地址落文件不落命令行，文件进 `.gitignore`。
  **绑定里有 `tier: production` 机器时必填**，不给即拒绝启动（`vigilNotConfigured`、
  `retryDisposition: MANUAL_REVIEW`）；全是 lab tier 时可省，Result 信封报一条 note。
  **不设 `--no-vigil`**。
- `lab watch vigil-check`：**真的往群里发一条**并验证整条链（推送 → `vm01` → IM），**不能进 CI**。
  并进 `lab watch` 那一行，子命令表仍是**十二行**。
- `vigilNotConfigured` 进 `src/Lab.Judge/reason-codes.json`，`layer: kernel`、`definedBy: "#30"`。
- **Vigil 报出的那条不是第 11 类事件。**十类是 Watch 检出、经 Courier 送的；这一条由一个不属于
  lab 运行时的东西发出，十类的定义不动。账里也不会有它——账在控制端，而控制端正是死的那个。
- `registry.json` 不动：**Vigil 不是角色**，不进登记册、不占 `tags`。

## 11. 被实测关掉的两个候选

**探针反向判定**：`#30` 正文写的理由「它在车上、多半没有出网的路」已被 #28 翻掉（车有网）。真正
关掉它的是路由——从 `agv01` 实测（2026-09-09）**够不到 `vm01`**（`192.168.200.50:22` tcp=False），
够得到的只有 `factory01`（不能常驻）与控制端（正是被监控者）。**车上有网，却没有一个不需要外部
凭据的地方可送。**

**GitHub Actions 定时任务**：`vm01` 的机器级 `HTTP_PROXY`/`HTTPS_PROXY` 四个变量全部指向
`http://172.19.162.241:7890`——控制端的 Clash。**runner 到 GitHub 的路穿过被监控者**，控制端整机
死掉时 cron 根本不触发。这个看门狗会在最需要它的那一刻恰好失效。
