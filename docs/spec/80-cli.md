<!-- 来源：#16 收拢，形状归 #33 -->

# CLI

agent 与 lab 之间只有这一条接口。子命令名收拢于 #16；**参数、输出信封、退出码、`reasonCode` 的
命名空间归 [#33](https://github.com/trytoreachpeak0/8005-test-lab/issues/33)，尚未定。**

入口是仓库根的 `lab.ps1`：`pwsh <仓库>/lab.ps1 <子命令> ...`。不装 PATH shim。

| 子命令 | 定于 |
| --- | --- |
| `lab run <scenario>` | #6 #10 |
| `lab watch start` / `stop` / `status` / `grant` | #20 #15 |
| `lab scenario list` | #10 #22 |
| `lab machine check` | #5 #6 #12 |
| `lab adapter check` | #4 |
| `lab do <role> <verb>` | #4 |
| `lab capture verify` / `seal` | #11 #20 |
| `lab observation render-ledger` | #22 |
| `lab invariant list` / `check` / `replay` | #21 |
| `lab read <role>[/<instance>] <controlPlane\|tap>` | #6（原 `lab probe`，#16 改名） |
| `lab toolkit sync` / `verify` | #16 |

**明确不做**：`lab session open`（#6 拒绝，理由未变）。

**留给别的票**：`lab capture list` / `find`（#25）、`lab observation prune`（#26）。

## 两条已定的命名事实

- **`lab probe` 改名 `lab read`**：与 #20 的 Probe（探针）撞名。没选 `inspect`（#7 占，是内核 `ui`
  控制面暴露的 UIA 模式动词之一）也没选 `snapshot`（#4 占，是可选的生命周期钩子名）。`read` 还多说
  对了一件事——它只有读侧，正好覆盖 #23 定的 Tap。
- **`lab machine check` 与 `lab toolkit verify` 的分界**：前者核登记册的声明对不对，后者核 lab 自己
  放在那台机器上的东西还在不在、哈希对不对。Toolkit 不在登记册里。
