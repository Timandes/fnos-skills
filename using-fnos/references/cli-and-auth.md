# CLI、连接与授权参考

## 产品与命令对应关系

| 产品 | 实际入口 | 命名连接 | 使用方式 |
| --- | --- | --- | --- |
| 旧飞牛 `@trimjs/trim-cli` | `trim-cli` | profile | 若已安装，优先使用；以该版本帮助为准 |
| `Timandes/fnos-cli`，npm `fnos-cli` | `fnos` | server | 无 trim-cli 时首选；本技能默认安装对象 |
| 新飞牛 `@fnnas-labs/fnos-cli` | `fnos-cli` | profile | 已安装时可用；与上两者独立，不共享授权 |

不要将三个 npm 包、两个同名产品或它们的配置目录混为一谈。安装来源可核实：

```sh
npm view fnos-cli name version bin repository engines --json
npm install -g fnos-cli
fnos --version
fnos --help
```

这是安装工具，不代表已连接 NAS。不要自动升级或删除现有 trim-cli。用户明确选择新飞牛产品时按其当前安装文档执行，而不是安装同名的另一个 npm 包。

## 登录前版本探测与认证选择

先确认目标系统版本，再选择目标与客户端共同支持的认证方式。[飞牛官方 AI Agent 授权说明](https://help.fnnas.com/articles/v1/settings/authorize)要求 fnOS **1.2.0800 或更高版本**。

| 目标情况 | 下一步 |
| --- | --- |
| 已有对应设备的可用会话 | 通过最小只读请求核实身份、版本并复用会话；不重复登录 |
| fnOS 低于 `1.2.0800` | 使用兼容的传统登录或已有会话；不生成本次 AI Agent OAuth 链接 |
| fnOS 达到门槛，且固件与客户端文档或只读能力信息确认兼容 | 可主动启动该客户端的原生 OAuth 流程 |
| 版本或认证能力未知 | 继续只读探测；确实无法取得时请用户查看管理页信息，不默认支持 OAuth |

按数值段比较版本，例如 `1.1.31 < 1.2.0800 < 1.2.1000`；不要按字符串排序。预发布或无法解析的版本需另行核实，不能默认达到门槛。该门槛针对这里的 AI Agent 授权，不概括旧版所有 OAuth 用途，也不要求用户升级 NAS。

已有会话时可查询系统版本；没有会话时可用客户端支持的登录前只读元数据。`Timandes/fnos-cli` 0.5.0 所带 SDK 的 `FnosClient.connect()` 会在登录前取得 `hostName` 和 `trimVersion`。以下示例通过已安装 CLI 解析其 SDK，不读取凭据或发起登录；先检查实际版本和 API，按真实管理地址选择对应的 WS/WSS 协议及端口：

```sh
FNOS_CLI_ENTRY="$(command -v fnos)" \
FNOS_TARGET_ENDPOINT='wss://nas.example.com:5667' \
LOG_LEVEL=error node <<'JS'
const { realpathSync } = require('node:fs');
const { createRequire } = require('node:module');
const fromCli = createRequire(realpathSync(process.env.FNOS_CLI_ENTRY));
const { FnosClient } = fromCli('fnos');
(async () => {
  const client = new FnosClient();
  try {
    await client.connect(process.env.FNOS_TARGET_ENDPOINT, 8000);
    console.log(JSON.stringify({
      hostName: client.hostName, trimVersion: client.trimVersion
    }));
  } catch (error) {
    console.error(error.message);
    process.exitCode = 1;
  } finally {
    client.close();
  }
})();
JS
```

上例未适用于当前客户端或握手失败时，按实际帮助、包内文档寻找其他只读探测方式；连接故障不证明认证不兼容。管理首页 HTTP 200、本机 CLI 支持 OAuth、SDK 本地构造出 `/signin?...` URL，都不是目标 OAuth 能力探测结果。

## trim-cli：优先但不猜命令

先执行以下帮助查询，不依赖新版参数在旧版也存在：

```sh
trim-cli --version
trim-cli --help
trim-cli login --help
```

从帮助和同包文档找到 profile 元数据、连接目标、监控及只读身份查询。不要臆造 profile 列表命令，必要时仅投影文档指定配置中的名称和目标，避开 token 存储。

已核实旧版 `@trimjs/trim-cli` 0.1.1 的生产流程为交互式 OAuth。**仅在完成上述目标版本与能力检查后**执行以下登录；保持 PTY，让用户在该登录提示中输入 code，不能把 code 放进命令参数：

```sh
trim-cli --profile nas-10 --host nas.example.com --port 5667 --scheme https login --no-open
trim-cli --profile nas-10 system info
trim-cli --profile nas-10 monitor cpu
```

上例地址为示例，须替换成真实目标。登录成功后才执行后两条。仅当另一已安装版本的实际帮助及文档明确支持生产两段式流程时，才按下文“两段式授权”替换入口使用；不能因为缺少两段式登录就忽略可用的 trim-cli。

目标版本过低或不支持该 OAuth 流程时，不先尝试 OAuth 再等报错；核实包内文档并改用兼容的客户端，例如支持传统登录的 `fnos`。若客户端自身能力不足，同样说明限制后选择兼容客户端。不要复制旧 session 到新版，也不要为了尝试新客户端删除旧凭据。

## fnos：server、传统登录与交互 OAuth

命令示例中的 `nas-10` 是命名连接示例；`https://nas.example.com:5667` 是示例地址，执行前替换为已确认的真实 endpoint（保留实际协议及端口）。

```sh
fnos auth.status --help
fnos auth.status --server nas-10
fnos --raw sysinfo getHostName --server nas-10
fnos login --help
```

以上身份查询只在已存在对应连接时执行；没有连接时先做登录前探测。`auth.status` 只读本地元数据，不验证服务端；OAuth 业务请求可自动刷新。

### 低版本兼容登录

`fnos` 0.5.0 默认使用传统用户名/密码登录，显式参数为 `--auth password`；其实际帮助要求 `-u/--username` 与 `-p/--password`，没有交互式密码输入选项。低于 `1.2.0800` 的设备优先复用已有非 OAuth 会话；没有时，让用户在自己的本机终端按该版本帮助完成 `fnos login --auth password --server <目标名> --endpoint <真实WS/WSS地址>` 所需的用户名/密码参数。Agent 不要求用户回传密码，不代填真实密码到工具调用或文件；不把“没有 OAuth”误判为“只能升级 NAS”。

用户完成后执行 `fnos --raw sysinfo getHostName --server <目标名>` 及只读版本查询，核对身份和保存结果，再继续原任务。若账户要求两步验证，按当前 CLI 的实际帮助完成；不假定旧系统具有某种登录能力，登录失败也不改走已知不兼容的 OAuth。

### 已确认兼容的 OAuth 登录

仅在目标版本达到门槛且确认固件与客户端支持对应授权流程后执行：

```sh
fnos login --auth oauth --server nas-10 --endpoint 'https://nas.example.com:5667'
```

`fnos` 0.5.0 的 OAuth 登录需要 **交互 TTY**，没有 `--authorize-only` 或 `--code`。用执行工具的 `tty: true` 启动登录，保留返回的会话 ID；把真实授权链接交给用户，收到 code 后通过该会话的 stdin 输入并等到登录结束。登录可能调用系统浏览器；用户也可手动打开输出的链接。OAuth 使用 HTTP(S) origin，不能使用 WS(S) endpoint。先查看当前版本帮助，以支持后续版本变化。

若需枚举 `fnos` 已保存的 server，而当前版本没有枚举命令，可仅投影其设置文件中的名称、地址和认证类型。不要 `cat` 整个文件；下面程序只输出白名单字段，不回显密码、token 或 session：

```sh
python3 - <<'PY'
import json
from pathlib import Path
p = Path.home() / '.fnos' / 'settings.json'
if p.exists():
    data = json.loads(p.read_text())
    records = data.get('servers', {})
    if not records and 'endpoint' in data:
        records = {'default': data}
    for name, record in records.items():
        print(json.dumps({'server': name, 'endpoint': record.get('endpoint'),
                          'authType': record.get('authType', 'password')}, ensure_ascii=False))
PY
```

该文件路径针对 `Timandes/fnos-cli`；不要用它读取其他产品配置。若 CLI 版本或执行环境更换，先核对实际配置位置。

## 新飞牛 fnos-cli：两段式授权

优先读本机同版本附带的 connection/OAuth 文档及帮助：

```sh
fnos-cli --version
fnos-cli --help
fnos-cli login --help
```

完成目标版本与认证能力检查、确认当前客户端参数受支持后，使用真实目标与独立 profile。低版本目标不执行这组 OAuth 命令；当前客户端仅支持 OAuth 时，改用支持该目标的传统登录客户端：

```sh
fnos-cli --profile nas-10 --host nas.example.com --port 5667 login --authorize-only
# 用户自行在浏览器完成授权，主动回传一次性 code 后，保持相同参数换码。
fnos-cli --profile nas-10 --host nas.example.com --port 5667 login --code '<用户提供的code>'
fnos-cli --profile nas-10 --host nas.example.com --port 5667 system info
```

如果使用非默认 scheme、TLS 或配置选项，两次登录必须一致。检查输出中的 token 保存结果（版本提供时为 `token_persisted`、`refresh_token_persisted`），再执行只读验证。

pending 仅代表等待用户授权；一个 profile 在等待 code 时不重复生成链接。code 过期、pending 丢失或交换失败则停止使用旧 code。已经保存 token、仅后续探测失败时先只读验证，不再次消费 code。code 是敏感的一次性凭据，可能出现在聊天和工具记录中；不要复述或额外写文件。由 CLI 处理 token、PKCE verifier、安全访问码及受控存储。

## 示例：监控 CPU 1 分钟

存在 trim-cli 时先用 `trim-cli --profile nas-10 monitor cpu`（核对该版本帮助）。使用 `fnos` 时，在已确认并验证 `nas-10` server 后，单次 CPU 查询为：

```sh
fnos resmon cpu --help
fnos --raw resmon cpu --server nas-10
```

先检查一份真实响应，识别整机 CPU 百分比字段、单位（0–1 或 0–100）、时间戳，以及瞬时值或历史数组的含义。不能把单核相加当整机百分比，也不能将同一历史数组反复计入。

有原生流式监控则优先使用；否则用单调时钟控制完整 60 秒窗口，以约 5 秒为默认间隔，记录实际开始/结束时间，每次带时间戳保存成功响应。请求耗时计入周期，末端采样应覆盖至窗口终点。窗口内持续查询时不要每次重新授权。按有效样本统计；不规则间隔须注明平均值为样本均值，或依据接口区间做时间加权。缺测、历史窗口或服务端刷新频率导致的限制须如实报告。

trim-cli 与新飞牛 fnos-cli 的资源命令名可能不同，使用它们的监控帮助，不照搬 `fnos resmon cpu`。

## 维护来源

- [飞牛官方 AI Agent 授权说明](https://help.fnnas.com/articles/v1/settings/authorize)：该授权流程的 fnOS `1.2.0800` 最低版本要求。
- [Timandes/fnos-cli README](https://github.com/Timandes/fnos-cli)：npm 安装入口、server 语义、交互 OAuth、auth.status 与刷新。
- [飞牛 CLI 与 Skill](https://github.com/fnnas-labs/fnos-cli-skill)：新版命令、profile 和两段式 OAuth；以所装版本文档为准。
- 旧产品包名及入口可通过 `npm view @trimjs/trim-cli name version bin --json` 核实，完整命令以其本机帮助为准。
- trim-cli 0.1.1 的交互登录及监控命令已通过 npm 发布包内 README、SKILL 和 OAuth 参考核对。

以上方言核对于 2026-10-29；未来执行仍检查实际版本与帮助。
