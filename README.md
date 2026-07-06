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
| 定时 | 每周日 00:00~22:00 UTC，每 2 小时 (`0 */2 * * 0`) |
| 手动 | `workflow_dispatch` |

每周全量刷新，约 20 小时完成（10 组 × 2 小时间隔）。

## Pipeline / 流程

```
周日 00:00  cron #1   discover + Group 0  ── ~15 分钟
     02:00  cron #2   Group 1             ── ~10 分钟
     04:00          Group 2
     06:00          Group 3
     08:00          Group 4
     10:00          Group 5
     12:00          Group 6
     14:00          Group 7
     16:00          Group 8
     18:00          Group 9
     20:00  cron #11  merge → push → sync ── ~2 分钟
     22:00  cron #12  done → skip
```

进度通过 GitHub Actions **cache**（非 artifact）在 12 次 cron 之间传递，
包括 `state.json`、`repos.txt` 和 `partial/group-*.json`。当周日 cache 有效，
下周日 state.discoveredAt 过期 → 自动重新开始。

## Repos / 相关仓库

| 仓库 | 可见性 | 职责 |
|---|---|---|
| `github-actions-indexer` | 私有 | Go 源码 + Release 二进制 |
| `github-actions-catalog-update` | 公开 | 调度 workflow (本仓库) |
| `github-actions-catalog` | 公开 | 数据文件 `actions-versions.json` |
| `github-actions-latest` | 公开 | Claude Code skill，消费数据 |
