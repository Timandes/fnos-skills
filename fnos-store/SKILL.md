---
name: fnos-store
description: Use when a task concerns fnOS 磁盘、硬盘、SSD、NVMe、SMART 健康、存储空间、卷、存储池、RAID、容量、缓存、休眠或磁盘 I/O, including NAS storage analysis when the operating system needs confirmation. Excludes ordinary file browsing, Docker volume configuration, and non-fnOS storage execution.
---

# 飞牛磁盘与存储

用已验证的专用 CLI 查询和管理 fnOS 存储。先建立“设备 → 磁盘 → 存储空间/卷”的对应关系，再分析健康与容量。

## 入口与认证

**前置技能：using-fnos。** 直接命中本技能时，先读取相邻的 `../using-fnos/SKILL.md`，按其工具选择、设备绑定、版本探测和兼容登录流程完成连接；已有验证结果可复用。由 using-fnos 路由而来时不重新登录，也不再次路由。

认证流程只有 using-fnos 一处维护，不在这里另创登录方式。非 fnOS 设备按真实系统选择工具，不执行以下 fnOS 命令。

**没有目标 server/profile、会话过期或需要用户登录，都属于连接认证待完成，不是 CLI 缺少存储能力的证据。** 继续兼容登录流程，不因 SSH 可用或浏览器已登录而跳过 CLI。需要用户操作时给出具体步骤，等待期间只做不依赖认证的工作，不索取密码或 token。

## 查询顺序

先看所选客户端当前版本的存储帮助，再发业务请求。`Timandes/fnos-cli` 的包名是 `fnos-cli`，入口是 `fnos`；下表为本机 0.6.0 已核实的命令，其他客户端按自身帮助执行。每次业务调用显式指定目标 server/profile。

| 目的 | fnos 命令与参数 |
| --- | --- |
| 存储空间、卷及容量概况 | `store general` |
| 物理磁盘、型号、序列号及归属 | `store listDisk` |
| 每块盘的现有 SMART 信息 | `store diskSmart --disk <实际设备名>` |
| 阵列/存储状态 | `store state --name <实际阵列名>` 或 `--uuid <实际UUID>` |
| 磁盘 I/O | `resmon disk` |
| 缓存/休眠等专项信息 | `store --help` 中实际存在的对应只读命令 |

例如：已按 using-fnos 验证 `nas-9` 后，先确认帮助，再查询：

```sh
fnos store --help
fnos store diskSmart --help
fnos --raw store general --server nas-9
fnos --raw store listDisk --server nas-9
# 仅当列表确实包含 sda 时执行；其余磁盘逐一使用列表中的实际名称。
fnos --raw store diskSmart --disk sda --server nas-9
```

从响应提取真实阵列名/UUID 后查询 state，不猜名称或参数。按问题范围读取，普通磁盘健康分析覆盖全部相关物理盘；只问容量时不必额外读取所有 SMART。读写性能监控按 using-fnos 的采样和单位规则执行。

## 能力缺口与回退

只有以下证据才支持回退：当前帮助及同版本文档确认缺所需接口/认证方式，或真实请求返回经核实的系统不支持/接口缺失。先确认不是参数用错、网络故障、会话失效、scope 或账户权限问题。

回退前向用户简述“缺哪个指标或操作、哪项证据、改用什么工具”。按 using-fnos 选择兼容客户端、API、SSH 或已有管理页。CLI 能取得的部分先用 CLI 完成，其他工具只补缺口。用户明确指定工具时遵循用户要求。

| 容易误判的理由 | 应做的事 |
| --- | --- |
| 没有 nas-9 的会话，SSH 已能登录 | 按 using-fnos 完成目标 CLI 登录 |
| 帮助已经看过，可以直接去管理页 | 帮助只证明入口存在，接着发目标只读请求 |
| CLI 未给 NVMe 错误码或 Btrfs 统计 | 明确缺口后用其他工具补取，不重新用 UI 采集全部指标 |

## 分析与操作边界

- 分开说明物理盘健康、阵列冗余、文件系统状态、容量和 I/O。SMART 总体正常不能证明文件系统无错；单盘 Basic 即使底层显示 raid1，也不提供第二块盘的镜像冗余。
- 结合原始值、单位、型号和厂商定义解释 SMART。累计 CRC、错误日志、不安全关机次数不能单凭一次快照判断当前故障；厂商磨损字段和出厂坏块不能套用通用百分比或新增坏块解释。趋势结论需有可比的历史样本。
- 报告检查时间、磁盘与卷的映射、关键异常、证据限制及下一步。容量保留接口原单位或标明 GB/GiB、TB/TiB；远程挂载与本地磁盘分开。
- 查询健康默认只读。自检、scrub、修复、基准写入、TRIM、格式化、建池/删池、扩容和缓存配置均不从“分析状态”推导授权。用户明确要求管理操作时，核对目标盘序列号、空间归属、当前状态及授权范围，按真实接口执行并回查。
