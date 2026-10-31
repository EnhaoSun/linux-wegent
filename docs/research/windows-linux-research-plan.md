# 微信个人 Agent：Windows 机制拆解与 Linux 复刻研究计划

> 日期：2026-10-31｜状态：Windows 第一轮源码审计已完成；Linux 实机尚未验证。本文及后续项目文档统一使用中文，代码标识符、API 名称与源码路径保留英文。

## 目标与边界
先拆解 [wechatauto-replica](https://github.com/fanyuantaier/wechatauto-replica/blob/main/README.zh-CN.md) 的真实数据库读取链路，再以 [chatlog-keeper](https://github.com/labazhou2024/chatlog-keeper) 为基础，研究 Linux 官方微信的本地只读 Connector。

首个 MVP 仅做：本地历史消息查询、增量事件、Agent 检索接口。不做自动发消息、微信协议模拟、客户端补丁或隐蔽对抗检测。只处理账号持有者明确授权的数据；原始密钥、内存内容和聊天记录默认不上传云端。

## 第一阶段：Windows 机制审计
**交付：** [windows-mechanism.md](windows-mechanism.md)

1. 定位微信进程和正确的账号数据目录。
2. 分开研究两种取钥路径：`Config.Cipher` 被动内存扫描；旧版 `cfg` 主密钥恢复。不得将两者混为一谈。
3. 研究候选密钥与数据库第一页 HMAC 验证、不同数据库的盐和逐库密钥。
4. 研究解密缓存、密钥缓存、WAL、消息分片、发送者身份解析。
5. 研究消息监听的轮询与增量语义，补充测试证据。
6. 每项结论标记“源码可见 / README 声称 / 尚待实测”。

**验收门槛：** 从进程定位到新消息事件能讲清完整链路，并能找到对应源码入口。

## 第二阶段：跨平台抽象
**交付：** `portability-matrix.md`

| 模块 | Windows | Linux 复刻方向 |
|---|---|---|
| 进程发现 | `tasklist`、进程 PID | `/proc`、进程映射 |
| 内存读取 | `OpenProcess` + `ReadProcessMemory` | `process_vm_readv` |
| 密钥定位 | `Config.Cipher` 配置对象 | 需要重新研究 ELF/C++ 对象布局 |
| 密钥验证 | 数据库第一页 HMAC | 优先复用算法与测试 |
| 数据快照 | Windows 侧数据库缓存 | 复用 chatlog-keeper 快照能力 |
| 消息解析 | 多分库、联系人映射 | 对照 Linux Schema 验证 |
| 增量监听 | 数据库轮询 | inotify 触发 + 游标补查 |

## 第三阶段：Linux 基线研究
**交付：** `linux-gap-analysis.md`

重点审计：
- `chatlog_keeper/core/_linux_process_memory.py`
- `chatlog_keeper/scripts/linux_memory_scan.c`
- `chatlog_keeper/linux_key.py`
- `chatlog_keeper/core/_snapshot.py`
- `docs/linux.zh.md`

**已知边界：** Ubuntu 默认 Yama 策略可能拒绝读取现有进程；C helper 存在 ptrace 回退；Linux 4.1.13.9 的已记录成功案例使用 GDB 硬件断点，不能算“纯被动扫描成功”。

## 第四阶段：Linux 最小验证
**交付：** `linux-poc-results.md`

先在自写模拟进程与合成数据库中验证权限、内存读取、密钥验证和快照；再在明确授权的测试账号与官方 Linux 微信上评估只读兼容性。记录“权限拒绝 / 可读取但未定位密钥 / 密钥通过 HMAC 验证”三种结果。禁止将“可读内存”误判为“已能解密微信”。

不修改系统级 Yama 设置、不要求 root 绕过权限；不在第一版引入 Hook、注入、进程写入或非官方微信协议。

## 第五阶段：只读增量 Connector
**交付：** `docs/architecture/linux-connector.md`

- 数据库及 WAL 一致性快照。
- 按账号、消息分片、消息 ID 维护持久游标。
- 事件至少一次投递、去重、断线补查。
- 验证重启、checkpoint、分库新增、撤回、附件延迟写入。
- 本地 Unix Socket 或仅 loopback API；密钥仅本地保存。

## Go / No-Go 标准
- 官方微信客户端不被修改。
- 取钥方式、权限、版本与失败原因均有可复现记录。
- 解密密钥经过真实目标数据库 HMAC 校验。
- 消息不会跨账号串库；多分片身份正确。
- 断线恢复后不漏消息、不重复对外事件。
- 不能证明零封号风险；必须记录政策、隐私与安全边界。

## 参考源码
- [Windows 中文 README](https://github.com/fanyuantaier/wechatauto-replica/blob/main/README.zh-CN.md)
- [Windows 数据库引擎](https://github.com/fanyuantaier/wechatauto-replica/blob/main/wechatauto/db.py)
- [Linux 取钥与安全模型](https://github.com/labazhou2024/chatlog-keeper/blob/main/docs/linux.zh.md)

> 说明：这是一份研究与验收计划，不代表 Linux 被动取钥已成功，也不代表商业部署的账号安全性已验证。
