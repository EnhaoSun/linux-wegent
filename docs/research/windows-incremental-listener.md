# Windows 消息增量监听机制：第二轮源码审计

> 2026-10-31；静态源码审计。上游源码树：`bc2e6f2461330eb1a53996d2f2afc18416376b52`。重点文件 `wechatauto/db.py`、`wechatauto/wx.py`、`wechatauto/demo_listen.py`。尚未进行 Windows 实机验证。

## 核心结论

Windows 的新消息监听是**本地数据库增量轮询**，不是监听微信网络协议或在微信收消息函数上安装 Hook。数据库解密与 WAL 合并是前置步骤，监听器对解密后的 SQLite 视图做查询。

## 源码调用链

```text
WeChat.StartListening()
  → wx.py:_listener_start()
  → db.Listener(db, interval=WxParam.LISTEN_INTERVAL)
  → Listener.start()
  → Listener._run()
  → Listener._poll_once()
      → db.get_sessions(limit=500)  # 全局监听模式下发现会话
      → db.get_new_messages(user, since_seq=...)
          → _run_msg_query()
          → _msg_conns()
          → _message_dbs()  # 动态发现消息分片
          → _open() / _open_unlocked()  # 更新解密缓存及 WAL 视图
      → Listener._dispatch()
      → Listener._worker_run()
      → Listener._run_callbacks()
      → Listener._commit_watermark()
      → Listener.save_watermark()
```

参考：[wx.py 第 1455 行附近](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/wx.py#L1455)、[db.py 第 2944 行附近](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L2944)。

## 1. 轮询机制

`Listener.__init__(interval=1.0)` 的默认间隔是 1 秒；`demo_listen.py` 也使用 `Listener(db, interval=1.0)`。高层 `WeChat` 类实际使用 `WxParam.LISTEN_INTERVAL`，需另核对该常量的当前取值。

`_run()` 在独立线程循环调用 `_poll_once()`，每轮结束等待指定间隔。单轮异常会输出错误并继续下一轮，不会直接停止整个监听器。

**注意：** 这是“每轮完成后再等待”，不是精确固定的每秒一次；解密或数据库查询耗时会影响实际延迟。

## 2. 增量游标：sort_seq

`_poll_once()` 计算每个会话的起读点：

```python
since = max(watermark[user], inflight[user])
messages = db.get_new_messages(user, since_seq=since)
```

- `watermark`：已确认处理的最大消息序号。
- `inflight`：已经交给工作线程但尚未确认的最大序号。
- `get_new_messages`：查询比起读点更新的消息，默认每批最多 200 条。

源码：[db.py 第 2169 行附近](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L2169)、[第 3080 行附近](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L3080)。

**需要进一步验证的风险：** 如果 `sort_seq` 在不同分片间不满足严格全局单调，单一会话水位可能遗漏晚出现但序号较小的消息。Linux 版不能在未测试前照搬。

## 3. 并发模型

`_dispatch()` 为每个会话建立一个工作队列与串行线程。同一会话顺序处理，不同会话并行；轮询线程只负责发现与投递，不等待耗时 AI 回调完成。

这是一个适合 Agent 的设计，但大量活跃会话可能产生较多线程。Linux MVP 可以改为有界工作池与按会话分区的队列。

## 4. 水位持久化

默认水位文件位于 `db.workdir/listener_watermark.json`。`_load_watermark()` 在初始化时恢复进度；`save_watermark()` 写临时 JSON，再用 `os.replace` 原子替换。默认写盘节流间隔 1 秒，停止监听时强制保存。

**风险：** JSON 写盘异常在源码中可能被忽略；回调完成与水位落盘之间也存在崩溃窗口。因此无法直接承诺端到端 exactly-once。

## 5. 失败处理的重要问题

`_run_callbacks()` 对失败回调重试，默认最多重试 3 次（总计最多 4 次尝试）。但 `_worker_run()` 在重试耗尽后依然调用 `_commit_watermark()`。

**后果：** 对普通消息提醒，这避免一条坏消息永久卡住队列；对需要长期记忆的 Personal Agent，则可能造成消息永久漏处理。

Linux 版建议：先将消息事件与唯一 ID 持久化到本地队列，再推进采集游标；下游 Agent 处理失败只更新任务状态，保留重试或死信记录。外部效果使用幂等键去重。

## 6. 数据库缓存与 WAL

`_open_unlocked()` 比较主库与 WAL 的修改时间、大小及 stamp。主库变化或 WAL 重置时重建解密缓存；仅有增量帧时可更新视图。重建时使用临时文件、完整性检查及原子替换。

源码：[db.py 第 1387–1517 行](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L1387-L1517)。

源码存在明确降级策略：WAL 合并失败时可以使用仅主库的可读副本，因此**可读不等于已经读到最新消息**。Linux Connector 需要区分“数据健康”和“新鲜度”，并在 WAL 失败后重试补查。

`_message_dbs()` 每次检查消息分片目录，出现新分片时补提取密钥；`_msg_conns()` 对同一会话在多个分片中的消息表进行合并。

## 7. 全局监听和新会话

`add_all(callback, discover=True)` 初始化时从 `get_sessions(limit=500)` 注册已知会话，之后每轮继续扫描并注册新会话。单会话与全局回调可以并存。

**潜在边界：** `limit=500` 不是无限会话发现保证。Linux 版应设计全量索引与分页补扫，不能把“当前最近 500 个会话”当成完整会话集合。

## 8. 对 Linux Connector 的建议

1. 复用 `chatlog-keeper` 的本地解密和一致性快照基础。
2. 消息发现使用周期性增量 SQL；`inotify` 仅用于减少延迟，不作为唯一可靠事件源。
3. 保存 `account + shard + message identity` 去重键，验证 `sort_seq` 跨分片行为。
4. 将“采集成功入本地队列”和“Agent 消费成功”分开确认。
5. 使用持久化队列（例如 SQLite）而不是仅用内存队列与 JSON 水位。
6. 监控快照新鲜度、WAL 合并失败、积压队列、断线补查。
7. 不把读取微信进程内存作为常态轮询步骤；密钥验证后优先缓存受保护的密钥，后续读取本地数据库。

## 下一轮待验证

- [ ] 核对 `WxParam.LISTEN_INTERVAL` 的实际值。
- [ ] 逐行确认 `get_new_messages` 的跨分片排序与限制行为。
- [ ] 测试 WAL checkpoint 期间的消息完整性和延迟。
- [ ] 检查历史消息补采、撤回/编辑、删除等更新事件。
- [ ] Linux `chatlog-keeper` 的消息 Schema 与 Windows 是否兼容。
- [ ] 构造回调失败、进程崩溃、重复启动的测试矩阵。

## 参考源码
- [监听实现 db.py](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L2944)
- [增量查询 db.py](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L2169)
- [数据库缓存与 WAL db.py](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py#L1387)
- [高层监听 wx.py](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/wx.py#L1455)
- [监听示例](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/demo_listen.py)
