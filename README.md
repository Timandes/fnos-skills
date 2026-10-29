# fnos-skills

NAS 与飞牛 fnOS 操作技能集合，每个技能使用独立的同名目录。

## 安装

使用 [Skills CLI](https://github.com/vercel-labs/skills) 安装到 Codex 用户目录：

```sh
npx skills add Timandes/fnos-skills --skill using-fnos --global --agent codex
```

查看仓库中可安装的技能：

```sh
npx skills add Timandes/fnos-skills --list
```

省略 `--global` 可安装到当前项目。需要其他 Agent 时，按 Skills CLI 支持的名称调整 `--agent`。安装后在对话中调用 `$using-fnos`，或让 Agent 根据 NAS 任务自动选择。

## using-fnos

[技能说明](using-fnos/SKILL.md) · [CLI 与授权参考](using-fnos/references/cli-and-auth.md)

- NAS、飞牛 fnOS、资源监控、磁盘、文件、Docker、网络与系统管理相关任务均可触发。
- 优先使用 `trim-cli`；否则使用 `fnos-cli`，缺失时安装。npm 包 `fnos-cli` 的实际入口是 `fnos`，命名连接称为 server；飞牛另一个客户端 `@fnnas-labs/fnos-cli` 使用 `fnos-cli` 入口及 profile。
- 先核对目标设备、系统版本及认证能力，再选择兼容的登录方式。fnOS 低于 `1.2.0800` 时复用可用会话或使用兼容传统登录；达到门槛并确认支持后才发起 AI Agent OAuth。版本或能力未知时先探测，不直接生成 OAuth 链接。
- 用户自行完成本机终端登录或浏览器授权。仓库只分发工作流程，不包含 NAS 凭据或 session；每台执行主机按目标独立登录。

```text
using-fnos/
├── SKILL.md
├── agents/openai.yaml
└── references/cli-and-auth.md
```
