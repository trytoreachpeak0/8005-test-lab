# observations/

一段 Watch 的产物落在这里，一个目录一段观察：`<UTC 日期>-<被测系统名>-<watchId 后 6 位>`。
第二段是被测系统名而不是 scenario 名，因为 watch 没有场景。形态由
[Watch 与 Observation 的形态](https://github.com/trytoreachpeak0/8005-test-lab/issues/20) 定。

**目录内容不进 git**，`.gitignore` 只放行这份 README。这里是 `--observation-root` 的默认值。

一段 watch 没有终点，所以它按 UTC 日与字节双条件切片：切走的分片立刻封存，**当前分片永远是
`active` 而不是 `unsealed`** —— 那是它正常工作的样子。人可读的账 `ledger.md` 在观察根滚动追加，
不在分片里。

Observation 的保留与检索归
[#26](https://github.com/trytoreachpeak0/8005-test-lab/issues/26)。
