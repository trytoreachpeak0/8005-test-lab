# 规格

这是 `8005-test-lab` v1 的规格，destination 说的那份「能交给 to-spec → to-tickets → implement 的
东西」。读者是实现者。

## 它与票的关系

**票是权威，规格是派生。** 与 `ledger.md` 是渲染产物、jsonl 才是权威数据同构
（[#22](https://github.com/trytoreachpeak0/8005-test-lab/issues/22)）。区别是规格没法自动生成——票是
散文——所以靠一条工作流规则顶上：

> **一张票关闭时，同一次收尾里更新 `docs/spec/` 的对应文件。**

这条与「写决议评论、关票、更新地图 Decisions so far、graduate 该毕业的雾」并列，是收尾的第五件事
（[#16](https://github.com/trytoreachpeak0/8005-test-lab/issues/16) 定）。

**规格里不重述推理，只写结论。** 推理是票的价值，而这条链上的下一环要的是「做什么」，不是「为什么
当初排掉了另外两个方案」。每份文件头一行 `<!-- 来源：#4 #10 -->`，改票时能反查。

## 当前状态

`docs/spec/` 建于 2026-09-08，那时地图上已经有十六张关闭的票。**它们的结论尚未回填**——每份文件
现在只有该主题的要点清单，完整规格待回填，进度见
[#35](https://github.com/trytoreachpeak0/8005-test-lab/issues/35)。
`90-repository-layout.md` 与 `00-overview.md` 是完整的，`95-self-test.md`（#29）、`80-cli.md`（#33）、
`65-courier.md`（#28）与 `66-vigil.md`（#30）也是——后四份是关票时按即时规则写的。

新关闭的票按上面那条规则即时写入，不进回填队列。

## 分片

| 文件 | 来源票 |
| --- | --- |
| `00-overview.md` | 地图 #1 |
| `10-registry-and-binding.md` | #5 |
| `20-adapter-contract.md` | #4 #10 |
| `30-transport.md` | #6 |
| `40-deployment.md` | #12 |
| `50-capture-and-evidence.md` | #11 |
| `60-watch-and-observation.md` | #20 #22 |
| `65-courier.md` | #28 |
| `66-vigil.md` | #30 |
| `70-judgement.md` | #21 #15 |
| `80-cli.md` | #16 收拢，形状归 #33 |
| `90-repository-layout.md` | #16 |
| `95-self-test.md` | #29 |

`10`~`80` 讲的是 lab 要造的那个东西；`90` 与 `95` 讲的是 lab 自己这个仓库。
