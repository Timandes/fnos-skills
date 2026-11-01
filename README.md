# fnos-skills

NAS 与飞牛 fnOS 操作技能集合，每个技能使用独立的同名目录。

## 安装

使用 [Skills CLI](https://github.com/vercel-labs/skills) 安装到 Codex 用户目录：

```sh
npx skills add Timandes/fnos-skills --skill using-fnos fnos-store --global --agent codex
```

查看仓库中可安装的技能：

```sh
npx skills add Timandes/fnos-skills --list
```

省略 `--global` 可安装到当前项目。需要其他 Agent 时，按 Skills CLI 支持的名称调整 `--agent`。安装后在对话中调用 `$using-fnos` 或 `$fnos-store`，或让 Agent 根据 NAS 任务自动选择。两个技能应同时安装：using-fnos 负责连接与认证，存储任务路由到 fnos-store。

## using-fnos

[技能说明](using-fnos/SKILL.md) · [CLI 与授权参考](using-fnos/references/cli-and-auth.md)

- NAS、飞牛 fnOS、资源监控、磁盘、文件、Docker、网络与系统管理相关任务均可触发。
- 磁盘、硬盘、SSD、SMART、存储空间、卷、RAID、容量及磁盘 I/O 任务路由到 fnos-store；没有目标 CLI 会话时先完成兼容登录，不能仅因 SSH 或浏览器可用就回退。
- 优先使用 `trim-cli`；否则使用 `fnos-cli`，缺失时安装。npm 包 `fnos-cli` 的实际入口是 `fnos`，命名连接称为 server；飞牛另一个客户端 `@fnnas-labs/fnos-cli` 使用 `fnos-cli` 入口及 profile。
- 先核对目标设备、系统版本及认证能力，再选择兼容的登录方式。fnOS 低于 `1.2.0800` 时复用可用会话或使用兼容传统登录；达到门槛并确认支持后才发起 AI Agent OAuth。版本或能力未知时先探测，不直接生成 OAuth 链接。
- 用户自行完成本机终端登录或浏览器授权。仓库只分发工作流程，不包含 NAS 凭据或 session；每台执行主机按目标独立登录。

```text
using-fnos/
├── SKILL.md
├── agents/openai.yaml
└── references/cli-and-auth.md
```

## fnos-store

[技能说明](fnos-store/SKILL.md)

- 覆盖飞牛磁盘、SSD、NVMe、SMART、存储空间、卷、存储池、RAID、容量、缓存、休眠与磁盘 I/O。
- 复用 using-fnos 的设备绑定、版本探测和认证流程，使用当前 CLI 帮助确认存储命令与参数。
- 分别评估磁盘与存储空间：先按型号及协议区分 U.2/NVMe/SATA SSD、SAS/SATA HDD；普通 HDD 重点看 SMART 05，SAS HDD 看 G 表，SSD 看健康度；类型未知但 SMART 明确提供 05 重分配计数及原始值时，可据此判断介质状况，非零判警告，零值不单独证明健康。所有磁盘温度不高于 50°C。
- 每块磁盘必填仅评价介质的磁盘状态（不可用 / 警告 / 健康 / 无法判定）；介质证据不足时为无法判定，链路、掉电历史和温度另列，不改变介质定性。关注范围内每个空间必填容量状态（不可用 / 警告 / 充足），并附判定依据。
- 仅关注目标服务器本地存储；远程 SMB/CIFS、NFS 等挂载不查询、不报告、不告警。本地数据卷及系统根文件系统的剩余容量须同时不低于 3% 和 3GB；EFI、独立 `/boot`、swap 等引导或交换专用分区不纳入容量表或阈值判定；RAID 另查降级及重建状态，容量结论不代替阵列健康。只有核实具体能力缺口后才用其他工具补充。
- 普通文件浏览、Docker volume 配置及非 fnOS 存储执行不进入此技能；状态分析不包含自检或配置修改。

```text
fnos-store/
├── SKILL.md
└── agents/openai.yaml
```
