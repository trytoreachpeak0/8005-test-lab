# evidence/

一次 Run 的产物落在这里，一个目录一次 run：`<UTC 日期>-<scenario>-<runId 后 6 位>`。
形态由 [证据模型与多机回捞](https://github.com/trytoreachpeak0/8005-test-lab/issues/11) 定。

**目录内容不进 git**，`.gitignore` 只放行这份 README。这里是 `--evidence-root` 的默认值，
换个地方放就传那个参数。

`8005-agv-control-server/evidence/l2/` 那 74 份历史证据留在原处不动
（[L2 迁移边界](https://github.com/trytoreachpeak0/8005-test-lab/issues/17)），
`lab invariant replay --legacy-l2` 读得了它们。

证据的检索、保留与跨仓引用归
[#25](https://github.com/trytoreachpeak0/8005-test-lab/issues/25)。
