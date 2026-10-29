---
name: using-fnos
description: Use when a task concerns a NAS, 飞牛 fnOS, NAS CPU or resource monitoring, disks, storage, files, Docker, networking, system administration, or NAS CLI setup and authorization.
---

# NAS 与飞牛 fnOS 操作

NAS 相关任务均可启用本技能。先发现专用 CLI、确认目标设备及授权，再执行任务；SSH 登录失败不能用来判断 NAS 管理接口不可用。其他品牌 NAS 也先确认系统类型，不将 fnOS 命令强加给非 fnOS 系统。

## 1. 选择工具

先检查命令是否存在，再对存在的入口执行 `--version`、`--help`，确认其确实是 fnOS 客户端。

```sh
command -v trim-cli
command -v fnos
command -v fnos-cli
```

按以下顺序选择可用工具：

1. 有 `trim-cli` 时优先使用它，包括它已有的 profile 和授权流程。
2. 否则使用 `fnos-cli`。注意：`Timandes/fnos-cli` 的 npm 包名是 `fnos-cli`，实际命令是 **`fnos`**，命名连接称为 **server**；另一产品 `@fnnas-labs/fnos-cli` 的实际命令才是 **`fnos-cli`**，使用 **profile**。二者都存在时优先前者。根据实际帮助选择对应方言，不混用参数或凭据。
3. 都没有时，核实 npm 包来源后安装 `fnos-cli`（`npm install -g fnos-cli`，安装后检查 `fnos --version` 和 `fnos --help`）。缺少 Node/npm 时先检查环境和受支持的安装方式。安装失败须说明原因，不能直接跳过专用工具而索要 SSH 密码。

不要为了避免授权而切换到低优先级工具。所选 CLI 确实缺少任务所需能力时，先核实帮助和文档，再选择能完成该任务的客户端、API 或 SSH；说明选择依据。用户明确指定工具时遵循用户要求。

命令、安装来源、连接枚举和不同授权方式见 [CLI 与授权参考](references/cli-and-auth.md)。首次使用某种方言或遇到授权问题时读取该参考。

## 2. 绑定目标及验证授权

查询用户指定 NAS 的已有连接资料；本机有知识库时优先检索其索引及相关记录。将用户设备名对应到实际管理 endpoint、profile/server 和已知身份，后续调用始终显式选择该连接。

- 已有对应记录：检查非敏感元数据，核实 endpoint，然后执行一次该设备的只读请求。
- 无对应记录：先从已知资料获取管理地址；只有地址确实未知时才向用户询问。以设备名创建独立 profile/server，并**立即启动该 CLI 支持的授权流程**。
- 名为 `default` 的连接只有在 endpoint 已确认属于目标 NAS 时才能复用；不能因为它是唯一记录就默认目标正确。保留其他 NAS 的连接。
- 本地状态或 token 到期时间不证明服务端授权有效或失效。先让正常只读请求触发 CLI 自带刷新；刷新成功则继续。确认刷新被拒、无有效凭据、会话不可恢复或缺少所需授权 scope 时，启动重新授权。
- DNS、超时、证书、限流和连接失败应按连接问题诊断，不反复重新授权。权限不足时区分缺 scope、账户权限及网关不支持；仅在重新授权能解决时重走流程。

## 3. 主动启动授权

授权是任务的前置步骤，缺 profile/server 或授权不可用时直接发起，无需再问“要不要授权”。需要用户参与时展示实际生成的链接或当前终端，简明说明待完成的浏览器操作。

- 使用所选 CLI 的原生 OAuth 流程。用户在浏览器自行登录并确认授权；Agent 处理用户主动提供的一次性 code，或让用户在 CLI 终端输入。
- 支持两段式时先 `login --authorize-only`，保留 pending 状态；收到 code 后以同一客户端、主机、系统用户、配置目录、目标和 profile 执行 `login --code`。
- 仅支持交互登录时启动持续存在的 PTY，提供其生成的授权 URL，保持该会话等待 code。不能臆造 `--authorize-only`／`--code` 参数，也不能关闭会话后把 code 交给新会话。
- 不索要 NAS 密码或 access/refresh token，不输出凭据文件。用户拒绝或取消授权即停止依赖授权的工作；code 过期或换码失败时解释并重新生成流程，不循环重试旧 code。
- 授权成功须确认保存完成，再以同一 profile/server 做只读验证，并核对设备身份。只有实际请求成功才宣称已连接，随后继续原任务。

## 4. 执行及汇报

通过已验证的 CLI 完成用户要求；具体子命令以当前版本 `--help` 为准。NAS 监控使用资源监控接口。持续监控须覆盖完整时长，保留每次成功采样的时间、原始指标及单位；失败不按 0% 计入。结束后报告采样时段、有效样本数、平均值、最低值和峰值，说明指标是整机还是单核、缺测及接口限制。参考文档给出 1 分钟 CPU 监控示例。

管理操作按用户已授权的范围执行，并通过结果或回查验证。授权和只读验收本身不包含重启、改配置、磁盘自检或格式化。技能及分发包仅含工作流程，不携带任何设备的 session、密码或授权码。
