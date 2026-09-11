# github-actions-catalog-update

Weekly pipeline that builds the [`github-actions-catalog`](https://github.com/byte-me-labs/github-actions-catalog)
dataset — a JSON index of **23,000+ GitHub Actions** from the Marketplace.

每周构建 [`github-actions-catalog`](https://github.com/byte-me-labs/github-actions-catalog)
数据集的流水线 — 来自 Marketplace 的 **23,000+ Actions 的 JSON 索引**。

## Architecture / 架构

```
本仓库 (调度)
  │ 每 2h cron (周日 00:00–22:00)
  │
  ├─ checkout: github-actions-catalog
  ├─ download: github-actions-indexer binary (private)
  ├─ restore/save: GitHub Actions cache (state + partial files)
  ├─ run: update-actions-linux-amd64 -group-count 10
  └─ push: actions-versions.json → github-actions-catalog
```

## Schedule / 调度

| 触发 | 规则 |
|---|---|
| 定时 | 每周六、周日 UTC 每 2 小时 (`37 */2 * * 6,0`) |
| 手动 | `workflow_dispatch` |

一轮全量刷新需要 **10 次运行**（10 组，每次处理一组）。周六的投递量通常够
跑完一轮；周日全部 cron 是兜底——周六若因丢投递没跑完，周日接着跑，否则
每次运行在 1 秒内结束。

## Pipeline / 流程

```
周六 00:37  cron #1    discover + Group 0   ── ~10 分钟
     02:37  cron #2    Group 1              ── ~3 分钟
     04:37          Group 2
     06:37          Group 3
     08:37          Group 4
     10:37          Group 5
     12:37          Group 6
     14:37          Group 7
     16:37          Group 8
     18:37  cron #10   Group 9 → merge → push ── ~3 分钟
     20:37  cron #11   done → skip          ── <1 秒
     22:37  cron #12   done → skip          ── <1 秒
周日          全部 cron                        ── done → skip，<1 秒
```

进度通过 GitHub Actions **cache**（非 artifact）在多次 cron 之间传递，
包括 `state.json`、`repos.txt` 和 `partial/group-*.json`。

状态按**周期**而非日历日推进（`State.ShouldDiscover`）：跨过 UTC 零点后，
下一次运行继续当前周期，不再重新 discover。已完成的一轮在 6 天内保持新鲜
（`DiscoverAfterDays`），下一个周六才开新的一轮；未完成的周期若停滞超过
3 天（`StaleAfterDays`）则判定为卡死，重新开始。

## Repos / 相关仓库

| 仓库 | 可见性 | 职责 |
|---|---|---|
| `github-actions-indexer` | 私有 | Go 源码 + Release 二进制 |
| `github-actions-catalog-update` | 公开 | 调度 workflow (本仓库) |
| `github-actions-catalog` | 公开 | 数据文件 `actions-versions.json` |
| `github-actions-latest` | 公开 | Claude Code skill，消费数据 |
