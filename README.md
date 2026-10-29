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
- 先核对目标设备和只读访问；缺少目标连接或授权不可恢复时，主动发起原生 OAuth 授权。
- 用户自行完成浏览器登录与授权。仓库只分发工作流程，不包含 NAS 凭据或 session；每台执行主机按目标独立授权。

```text
using-fnos/
├── SKILL.md
├── agents/openai.yaml
└── references/cli-and-auth.md
```
