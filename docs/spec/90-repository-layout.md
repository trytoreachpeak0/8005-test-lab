<!-- 来源：#16 -->

# 仓库布局与工具链

## 1. 目录

```
8005-test-lab/
├─ CLAUDE.md                    agent 规则
├─ CONTEXT.md                   词汇表
├─ README.md                    人读的入口
├─ registry.json                机器登记册（#5）
├─ lab.ps1                      CLI 唯一入口
├─ src/
│  ├─ Lab.Judge/                判据、时间算子、安全门、Grant —— 纯函数，无 I/O
│  ├─ Lab.Core/                 控制端：登记册、绑定、传输、适配层加载、编排、证据、CLI
│  ├─ Lab.Remote/               run 作用域的远端半边（#6）
│  └─ Lab.Probe/                watch 作用域的探针（#20）
├─ adapters/
│  └─ <name>/                   一个适配层一个目录（#4）
│     ├─ adapter.json
│     ├─ roles/<role>/          固定文件名钩子
│     ├─ verbs/
│     ├─ scenarios/             <名字>.scenario.psd1 + <名字>.ps1（#10）
│     ├─ shared/                能被 lab 之外调用的纯判定（第 5 节）
│     ├─ <Name>.Adapter.psd1    断言辅助模块，认 $Context
│     └─ docs/onboarding.md     接入文档
├─ vigil/
│  └─ Watch-LabVigil.ps1        看着控制端的那一半，自足单文件，不加载上面任何模块（#30）
├─ tests/
│  ├─ unit/                     无外部依赖，任何机器
│  ├─ machine/                  要一台真机器才成立的
│  └─ fixtures/                 回放素材与假 adapter（不 gitignore）
├─ tools/
│  └─ toolkit.json              第三方工具清单，二进制不进 git（第 4 节）
├─ docs/
│  ├─ spec/                     本目录
│  └─ research/                 研究文档，带快照声明
├─ evidence/                    gitignore，留 README.md（#11）
└─ observations/                gitignore，留 README.md（#20）
```

## 2. 四个模块，分界是「在哪台机器上加载」

不按功能切。这条约束来自 #21 与 #15：判据必须能在被观察机器上离线求值，安全门必须能在那里判完 ——
那意味着有一块代码要同时住在控制端和探针里，而探针不能把整个控制端拖过去。

| 模块 | 加载在哪 | 装什么 | 不许依赖 |
| --- | --- | --- | --- |
| `Lab.Judge` | 控制端、`Lab.Remote`、`Lab.Probe` 三处 | 判据求值器、时间算子、四值结果、安全门判定、Grant 到期判定 | 其余三个模块；任何 I/O；任何传输 |
| `Lab.Core` | 只在控制端 | 登记册、绑定校验、传输、适配层加载与校验、编排、Capture 与证据、CLI | — |
| `Lab.Remote` | 被测机器（run 作用域） | 钩子调用 wrapper、NDJSON 分帧、本机读原语、`workRoot` 管理 | `Lab.Core` |
| `Lab.Probe` | 被观察机器（watch 作用域） | 采样环、修复执行、本机攒事件与补传、叫停哨兵检查 | `Lab.Core` |

**`Lab.Judge` 不许依赖别的三个**，是最硬的一条：回放就是把采样环的输入换成一个证据目录，而那只有
在求值器不认识「机器」「传输」「控制端」这些东西时才成立。它同时是唯一一个能被完整单元测试、一台
机器都不需要的模块。

模块内部按面切多个 `.psm1`，用 psd1 的 `NestedModules` 串起来，对外仍是一次 `Import-Module` ——
远端半边的 `Import-Module` 只做一次，之后每次调用都是远端内存里的函数调用。

**`vigil/Watch-LabVigil.ps1` 有意不在这张表里**（#30）。它跑在一台不是控制端的机器上，看着控制端
自己，所以它**不 `Import-Module` 上面任何一个模块**、不读 `registry.json`——依赖 lab 就意味着 lab
仓库的一次坏改动会同时干掉被看的和看着的，而那次失败不出声。它因此重复了 `Lab.Core` 里几十行
webhook POST 与 body 判定，**这份重复是故意的**：第 5 节那条「被 lab 之外调用的适配层判定必须是纯
函数模块」要消除分叉，这一条要消除共同故障，方向相反而考虑同源。详见 `66-vigil.md` 第 8 节。

## 3. lab 是仓库内的模块，靠路径导入，不发布

`Import-Module <仓库>/src/Lab.Core/Lab.Core.psd1`。没有打包、没有 gallery、机器上不
`Install-Module`。psd1 换来三样东西：

- `FunctionsToExport` —— #4 那条「内核原语不认任何被测进程名」的分界，唯一能被机器核对的地方。
- `RequiredModules` —— 把 Pester 的版本下限写死。
- `ModuleVersion` —— 让证据说得出用的是哪一版。

CLI 入口是仓库根的 `lab.ps1`，**不装 PATH shim**（那是在机器上留一样东西）。

**lab 自己的版本进证据**：`labVersion: { commit, dirty, dirtyFiles[], moduleVersion }`。
**脏工作树不拦**，但 `dirty: true` 时 `SUMMARY.md` 出现一条内核渲染的 caveat，且这份证据不能用来
支持 destination 的三条完成标准。工具链的可复现性与被测系统的判定是两类事，前者不改红绿。

## 4. Toolkit：lab 自己依赖的第三方工具

**二进制不进 git，唯一定义处是 `tools/toolkit.json`**（版本、文件级 sha256、上游 URL、许可证、
`verifiedBy`、`upgradeRequiresFieldTest`）。沿用工作区「只留配方不留制品」的惯例。

**到达三跳**：控制端取回并校验（只有控制端连得上上游）→ 经 lab 自己的传输送到目标机 → 目标机复验
sha256 再用。

**按 sha256 缓存**，落 `<workRoot>/toolkit/<name>/<sha256 前 12 位>/`，跨 run 跨 watch 存活。键取
sha256 而不是版本号：版本号可以被重新发布，sha256 不能；直接后果是升级即并存，回退只是改一行清单。

`workRoot` 下因此有四类生命周期不同的东西：

| 是什么 | 生命周期 |
| --- | --- |
| lab 的远端半边（`Lab.Remote`） | run 作用域，内容哈希同步，不清理 |
| 探针（`Lab.Probe`） | watch 作用域，部署-跑-清除 |
| 被测二进制的 stage 与安装 | 安装级，跨 run 存活，故意留下 |
| Toolkit | 按 sha256 缓存，跨 run 跨 watch 存活 |

**这张表被 #31 换掉了。** 它只覆盖 `workRoot` 之内，而 lab 在一台机器上留下的东西不止在
`workRoot` 里（桌面快捷方式、机器级环境变量、计划任务、`vm01` 上的 Vigil 都在外面）。
完整的清单、谁清、什么时候清，见 **`45-footprint.md`**：它改按两维分——**这是谁的东西**
（决定要不要过 #15 的门）与**它会不会自己消失**（决定要不要淘汰策略）。上面这四类在那张表里
分别落在不同的格子。

**升级的唯一入口是改 `toolkit.json`**，改它必须更新 `verifiedBy`。要不要先上机器重测一遍，判据是
**这个工具坏掉的时候会不会不出声**：`winapp ui` 换版本必须重跑锁屏实测（`--capture-screen` 在锁屏下
返回退出码 0 加一张锁屏壁纸，静默给错答案）；Pester 换版本不必（它坏了是响的）。

**`WINAPP_CLI_TELEMETRY_OPTOUT=1` 设在每次调用的进程环境，不设机器级** —— 机器级是足迹，在生产车
上还要过安全门，而 lab 是那些机器上唯一的调用方。

**Pester 的坑，三处落点**：`Lab.Core.psd1` 的 `RequiredModules` 写下限；测试入口显式按路径导入
Toolkit 里那一份；加载期检查主版本小于 5 就当场失败（`pesterVersionTooOld`）。三台机器实测都带着
Windows 自带的 Pester 3.4.0 且都在 `PSModulePath` 里。

**「哪台机器要哪个工具」不进 `registry.json`** —— 登记册只声明机器能力。它是从绑定推出来的：一个
角色声明了 `ui` 控制面，承载它的机器就需要 winapp。开跑前的可达性预检那一步顺带核对并按需投递。
`lab machine check` 核的是登记册的声明，`lab toolkit verify --machine <名字>` 核的是 lab 自己放的
东西还在不在。

## 5. 能被 lab 之外调用的适配层判定，必须是纯函数模块

入 JSON 出 JSON，不认 `$Context`、不认 `runId`、不认证据目录、不发起任何传输。位置
`adapters/<name>/shared/`，独立 psd1/psm1，**不 nested 进适配层主模块**——外部调用方通常只要其中
一个判定。

第一个例子是 `WireToGate.Validation`：`remote-ops/onboard-hmi/scripts/06-deploy-onboard-hmi.ps1`
里那段逐条复刻 `SQCD.Agv.Infrastructure/Configuration.cs` 启动校验的代码抽到这里，`06` 反过来引用
它。`06` 找不到它就**硬失败并提示跑 `setup.ps1`**，不静默回退到自己那份旧副本。

代价：`remote-ops/` 从此依赖 `repos/8005-test-lab/` 存在。

## 6. 测试

`tests/unit/`（无外部依赖）、`tests/machine/`（要一台真机器）、`tests/fixtures/`（回放素材与假
adapter，不 gitignore）。用 Pester 5，**真车上也跑，不加额外限制**。

`tests/machine/` 里每个测试**声明它碰什么**（进程、端口、桌面、路径）。这不挡任何测试跑，它是为了让
「一次自测跑完之后那台生产车上多了什么」可核。

测什么、测到什么程度归 #29；CI 怎么跑归 #13。

## 7. 命名

- 目录、Markdown、JSON：小写 kebab-case（`file-naming-convention/` 治理本仓）。
- pwsh 源文件：PowerShell 社区惯例 —— 模块 `Lab.Core.psd1`/`.psm1`，脚本 `Verb-Noun.ps1`。那份规范
  第 3 节的注脚明确把源文件留给各语言自己的习惯。
- 例外三个：`lab.ps1`（命令名不是函数名）、`registry.json`、`adapter.json`。
- 函数前缀统一 `Lab`（`Wait-LabCondition`、`Wait-LabProgress`）。

工作区既有实践的分界照抄：编号的一次性运维步骤用 `NN-kebab.ps1`，可重复调用的工具用
`Verb-Noun.ps1`。lab 里没有第一类。

## 8. 落地顺序

**产品代码等地图清空**：`src/`、`tests/`、`adapters/`、`lab.ps1`。

**不是产品代码的现在就落**：`docs/spec/`、`docs/research/`、`tools/toolkit.json`、`.gitignore`、
`evidence/README.md`、`observations/README.md`。依据是既有的——`CONTEXT.md` 在地图没清空的情况下
已经改过十二次，因为词汇表不是代码，而规格、清单与 gitignore 是同一类东西。
