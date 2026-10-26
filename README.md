# backend-keepalive

GitHub Actions 定时探活：**每小时**自动 ping 一次《时空跃动》游戏后端，防止云托管平台因长期无访问而把项目休眠/暂停。

> 即使没有任何人打开网页，这个仓库也会自己跑起来。

## 它做了什么

| 项 | 说明 |
|---|---|
| 触发方式 | `schedule` 每小时一次（UTC 17 分）+ 手动 `workflow_dispatch` |
| 探活目标 | `GET /rest/v1/levels?select=id&limit=1`（游戏自身的只读健康检查接口，无副作用） |
| 失败重试 | 单次超时 20s，最多 4 次尝试，指数退避 3s/6s/9s |
| 心跳日志 | 每次结果追加到 `heartbeat.log`（保留最近 2000 行） |
| 自动提交 | 每次探活都 commit + push |

### 为什么要自动提交？

GitHub 官方规则：**仓库连续 60 天没有任何活动，`schedule` 触发器会被自动停用**，而且仅靠 workflow run 不算 activity。
所以这里每次探活都真实提交一次 `heartbeat.log`，让仓库永远"有活动"，定时任务才能一直跑下去。

### 为什么是 17 分而不是 0 分？

Actions 的 cron 用 UTC。整点（0 分）是所有定时任务的默认选择，高峰时段排队延迟可能到十几分钟。
错开到 17 分，延迟通常只有几秒。

## 手动验证

进入仓库 **Actions → Backend Keepalive → Run workflow**，立刻跑一次。
日志里能看到类似：

```
attempt=1 http=200 96ms body=[{"id":4}]
2026-10-26 11:17:04 UTC | status=ok | http=200 | 96ms | run=3
```

`status=fail` 时整个 job 会标红，并在日志里给出明确错误。

## 关于 API Key

工作流里用的是游戏前端 `v4.html` 里同一个匿名 anon key（本来就会随页面下发给每个浏览器），
只读、无写权限，放在 workflow 里不构成额外暴露。

## 本地等价程序

同目录行为与本地 `backend_pinger.py`（每 60 秒一次、常驻运行）完全一致，区别只在触发方式：
本地是进程常驻，这里是 GitHub 托管。
