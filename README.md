# radar-data

Radar Engine 的**证据归档**。这个仓库只存机器数据，不存代码。

代码、评分标准、机会文件在另一个仓库（`GuihaiHU/radar`）。分开是刻意的：**进了版本控制的文件看起来就像权威事实**，
如果把 `radar.db` 或每日报告和代码放在一起，接手的 Agent 很容易把一份过时的产物当成当前状态。

## 这里有什么

```
raw/<source>/<date>.jsonl     采集到的原始事件（只追加，envelope 格式）
reports/daily/<date>.md       每次运行的候选报告（操作证据）
```

**不在这里的：** `data/radar.db`。它是派生物，能从 `raw/` 完整重建，所以不需要备份——
备份一个可以重算的东西只会多出一个可能与事实漂移的副本。

## Raw 事件的格式（envelope）

```json
{
  "v": 1,
  "source": "hn",
  "source_id": "49808617",
  "kind": "comment",
  "url": "https://news.ycombinator.com/item?id=49808617",
  "created_at": "2026-09-25T14:32:11Z",
  "fetched_at": "2026-09-26T07:12:16Z",
  "matched_query": "incumbent_frustration",
  "matched_phrase": "i can't believe there's no",
  "payload": { "...上游字段原样保留..." }
}
```

设计要点：

- `payload` **保留上游原样，文本不做 unescape**。转换留在 normalizer 层，所以归一化一旦有 bug，
  修完还能重跑。如果这里存的是处理后的文本，这个能力就永久丢失了——而"DB 可丢、raw 是 trail"整个设计正依赖它。
- 只剔除了**可证明冗余**的字段（HTML 源的 `_highlightResult` 是正文的副本）。
- `v` 是信封版本；格式变化时可据此检测。
- `kind` 是自由字符串，约定取值：`comment | story | post | issue | pr`。

**凭证永远不得写进这里。** 任何 token、cookie、会话凭据一旦落进 raw，就会永久留在仓库历史里。

## 怎么恢复

```bash
git clone git@github.com:GuihaiHU/radar-data.git
rsync -a radar-data/raw/ /home/harmony/radar/raw/
cd /home/harmony/radar/tools
node run.ts --rebuild          # 数据与 DB 全部重建
node verify.ts                 # 对账：确认重建结果与归档一致
```

`run.ts --rebuild` 只依赖 `raw/` 与代码仓库，不需要网络。

## 怎么更新

由 `/home/harmony/radar/tools/backup.sh` 自动同步（cron 每日运行）：
先做恢复对账（`verify.ts`），对账不通过就不推送，避免把已经损坏的归档当成好备份推上来。

## 已知限制

- **Reddit / Semrush 接入后需要重新评估本仓库**：Semrush 的条款要求缓存不超过 30 天，
  而这里会长期保留；届时需要为这些来源设定保留策略或排除它们。
