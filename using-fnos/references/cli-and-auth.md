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

## trim-cli：优先但不猜命令

先执行以下帮助查询，不依赖新版参数在旧版也存在：

```sh
trim-cli --version
trim-cli --help
trim-cli login --help
```

从帮助和同包文档找到 profile 元数据、连接目标、监控及只读身份查询。不要臆造 profile 列表命令，必要时仅投影文档指定配置中的名称和目标，避开 token 存储。

已核实旧版 `@trimjs/trim-cli` 0.1.1 的生产流程为交互式 OAuth；保持 PTY，让用户在该登录提示中输入 code，不能把 code 放进命令参数：

```sh
trim-cli --profile nas-10 --host nas.example.com --port 5667 --scheme https login --no-open
trim-cli --profile nas-10 system info
trim-cli --profile nas-10 monitor cpu
```

上例地址为示例，须替换成真实目标。登录成功后才执行后两条。仅当另一已安装版本的实际帮助及文档明确支持生产两段式流程时，才按下文“两段式授权”替换入口使用；不能因为缺少两段式登录就忽略可用的 trim-cli。

若旧版缺 OAuth 能力或目标 fnOS 已不兼容，检查实际错误及包内文档，说明兼容性限制后改用支持当前设备的客户端。不要复制旧 session 到新版，也不要为了尝试新客户端删除旧凭据。

## fnos：server 与交互 OAuth

命令示例中的 `nas-10` 是命名连接示例；`https://nas.example.com:5667` 是示例地址，执行前替换为已确认的真实 endpoint（保留实际协议及端口）。

```sh
fnos auth.status --help
fnos auth.status --server nas-10
fnos --raw sysinfo getHostName --server nas-10
fnos login --help
fnos login --auth oauth --server nas-10 --endpoint 'https://nas.example.com:5667'
```

`auth.status` 只读本地元数据，不验证服务端；业务请求可自动刷新 OAuth。`fnos` 0.5.0 的 OAuth 登录需要 **交互 TTY**，没有 `--authorize-only` 或 `--code`。用执行工具的 `tty: true` 启动登录，保留返回的会话 ID；把真实授权链接交给用户，收到 code 后通过该会话的 stdin 输入并等到登录结束。登录可能调用系统浏览器；用户也可手动打开输出的链接。OAuth 使用 HTTP(S) origin，不能使用 WS(S) endpoint。先查看当前版本帮助，以支持后续版本变化。

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

确认参数受支持后，使用真实目标与独立 profile：

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

- [Timandes/fnos-cli README](https://github.com/Timandes/fnos-cli)：npm 安装入口、server 语义、交互 OAuth、auth.status 与刷新。
- [飞牛 CLI 与 Skill](https://github.com/fnnas-labs/fnos-cli-skill)：新版命令、profile 和两段式 OAuth；以所装版本文档为准。
- 旧产品包名及入口可通过 `npm view @trimjs/trim-cli name version bin --json` 核实，完整命令以其本机帮助为准。
- trim-cli 0.1.1 的交互登录及监控命令已通过 npm 发布包内 README、SKILL 和 OAuth 参考核对。

以上方言核对于 2026-10-29；未来执行仍检查实际版本与帮助。
