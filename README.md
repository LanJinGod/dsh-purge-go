# dsh-purge-go

> **占位符提示**：本文所有 `OWNER/dsh-purge-go`、`security@example.com` 都是占位符，发布前按文末[发布前替换清单](#发布前替换清单)替换。

把桌面版 **dsh-purge** 的「提示词注入」内核重做成一份**只使用 DSH 官方扩展点**的插件包：把一段正文顶到 system prompt 最前面、压掉会抢身份的三段官方文案，外加题材分类扩展层、地区门、拒答续跑、目标卡、23 条红队技能与一份红队预设。零运行时依赖，不改引擎文件、不解 asar、不重启宿主。

一句话定位：**给 DSH 引擎加一层「破甲」提示词——一份明文正文 + 一组官方钩子，手机 PRoot、SSH 远程主机、桌面版 DSH 装同一份代码。**

[![node](https://img.shields.io/badge/node-%3E%3D22.19-3c873a)](https://nodejs.org/)
[![DSH](https://img.shields.io/badge/DSH-0.2.1--rc.2_.._0.3.0--0-5b8def)](https://www.npmjs.com/package/@deepseek-ai/dsh)
[![license](https://img.shields.io/badge/license-AGPL--3.0-blue)](LICENSE)
[![deps](https://img.shields.io/badge/deps-0-brightgreen)](package.json)
[![CI](https://github.com/OWNER/dsh-purge-go/actions/workflows/ci.yml/badge.svg)](https://github.com/OWNER/dsh-purge-go/actions/workflows/ci.yml)

> 前四个是 shields.io 静态徽章，写死的是本包自己的事实。CI 徽章指向本仓库 `.github/workflows/ci.yml`，工作流推上去之前会显示 404，这是正常的。

## 1. 这是什么：插件包，不是 App

它**不是**独立 App、不是 Android APK、不是 npm 全局 CLI。它是一个 **DSH（DeepSeek Harness，Cordis 插件体系）插件 + bundle**，宿主必须是 DSH 引擎。

| 形态 | 是什么 | 本包里对应 |
|---|---|---|
| 插件包 | ESM 包，`"type": "module"`、`main: lib/index.js`，导出 `name` / `inject` / `apply` / `DEFAULTS` | 整个仓库 |
| bundle | `package.json` 的 `dsh.bundle.patch` 指向 `cordis.patch.yml`，被 profile 当成一个补丁层组合进插件树 | `cordis.patch.yml`（一条 `insert`） |
| profile 依赖 | 装在 `$DSH_HOME/profiles/<profile>/node_modules`，同时登记进该 profile 的 `dependencies` 与 `dsh.profile.bundles` | `dsh plugin add` 或自解压安装器做的事 |
| 宿主引擎 | DSH 本体，提供 `systemPrompt` / `tools` / `commands` / `dshGo` 服务 | **必需**，版本范围见 §3 |

据此可以排除三种常见误解：

- **不是独立 App**：没有自己的窗口或进程。手机上的状态条与面板，是 dsh-go 宿主读 `package.json` 的 `dshGo.contributes` 之后画出来的。
- **不是 Android APK**：没有 `AndroidManifest.xml`、没有 `.so`、不申请 Android 权限；它跑在引擎的 Node 里。
- **不是 npm 全局 CLI**：`package.json` 没有 `bin` 字段，`npm i -g` 之后也不会多出任何命令；唯一入口是 `apply(ctx, config)`。
- **不是桌面版 dsh-purge 的替代品**：见 §12，桌面版那套评分/规则/rewind/更新器不在本包。

## 2. 能做什么

| 能力 | 落点（官方扩展点） | 行为 |
|---|---|---|
| 正文注入 | `systemPrompt.section()`，段名 `dsh-purge`，`order: -100000`，`interpolate: false` | 正文取自 `$DSH_HOME/prompt-inject.md`，缺失时用包内 `assets/default-prompt.md`；可整体替换、可按模型选变体 |
| 顶到最前 | `system-prompt/assemble` waterfall | 把自有段（`dsh-purge*`、`dshpg-*`）重排到列表最前并保持内部相对顺序，其余段保序 |
| 压冲突段 | 同一个 waterfall | 默认丢弃 `harness:identity`、`deployment:persona-prefix`、`deployment:persona-suffix`；`dropConflicts: false` 可关 |
| 能力协议 | 段 `dshpg-capabilities`，`order: -99999` | 英文 `## Capability selection by task (mandatory)` 六条；`$DSH_HOME/dsh-purge/capabilities.md` 可整体覆盖 |
| 目标卡 | 段 `dshpg-objective`，`order: -99998` | 读 `$DSH_HOME/redteam/current` → `engagements/<id>/engagement.yaml`，写成「只打：X / 范围：… / 不打：其它」；没有当前靶标就返回空串 |
| 地区门 | `agent/pre-step`（`{ global: true, prepend: true }`，只看第 1 步） | 读本轮任务文本，识别目标主机/字面 IP；命中受限地区就写入一句禁止语并结束这一轮。字面 IP 走随包网段表离线判定，不发 DNS、不发请求 |
| 拒答续跑 | `agent/status`（`{ global: true }`） | 一轮结束、最后一条助手消息是逃避句、用户没追问、本轮未追过 → 追加一句续跑；默认每会话上限 2 次、冷却 60 s，账本在内存、按 `turn/end` 的 seq 去重 |
| 压 mnemon 记忆 | `agent/pre-step` 包装 | 滤掉 mnemon 插件每步塞进来的 `source.kind === 'plugin'` 消息；`stripMnemon: false` 可关 |
| 题材分类层 | 组装时渲染 | `assets/categories.json` 出厂四条（26–29），`$DSH_HOME/dsh-purge/categories.json` 为覆盖层，写进正文的两对标记区 |
| 红队资产 | 启动时同步文件 | `skills/redteam/*.md`（23 条）→ `$DSH_HOME/redteam/skills/`；`presets/redteam/` → `$DSH_HOME/.agent-presets/redteam/`（已有则保留，`REDTEAM_PRESET_REFRESH=1` 才覆盖） |
| 命令 | `commands` 服务 | 一个 `/purge`，子命令覆盖开关、正文、变体、分类、路径 |
| 工具 | `tools` 服务 | 5 个：`purge_status` / `purge_apply` / `purge_revert` / `purge_prompt` / `purge_category`，每个都带一张只读 dsh-go 工具卡（≤8 KiB） |
| 原生界面 | `dshGo` 桥接 | 1 条状态条 + 2 个面板（`purge` / `prompt`）+ 1 个加号工具（`body`）+ 2 个输入区按钮（`toggle` / `reload`） |

正文解析优先级（先命中先用）：`state.byModel[当前模型]` 指向的变体 → `state.variant` 变体（默认 `default`，指 `prompt-inject.md`）→ `$DSH_HOME/prompt-inject.md` → 包内 `assets/default-prompt.md`。

## 3. 运行环境

### 3.1 引擎

| 项 | 值 |
|---|---|
| 引擎 | DSH（DeepSeek Harness）`>=0.2.1-rc.2 <0.3.0-0`（`engines.dsh`），实测 `0.2.2-rc.1` |
| Node | `>=22.19`（`engines.node`），实测 `v24.19.0` |
| 模块形态 | ESM（`"type": "module"`），无 CJS 入口 |
| 使用的官方扩展点 | `systemPrompt.section()`、`system-prompt/assemble`、`agent/pre-step`、`agent/status`、`tools`、`commands`、`dshGo` |
| 组合方式 | profile 的 `dsh.profile.bundles` 列出本包 → 启动时应用 `cordis.patch.yml` 的 `insert` → 插件进插件树 |

顶层只声明 `inject = ['systemPrompt']`；`tools` / `commands` / `dshGo` 都用 `ctx.inject([...])` 局部等待——没有这些服务的宿主不会被拖成空等。

### 3.2 操作系统、容器、架构

| 项 | 实测 | 说明 |
|---|---|---|
| 系统 / 容器 | Android 15 + Debian GNU/Linux 13 (trixie) PRoot | 引擎与 Node 都在容器里，宿主是 DSHBox 手机本机引擎 |
| 内核 / 架构 | 6.6.118 aarch64 | 代码里没有原生二进制与 `.node`，理论架构无关；只实测了 aarch64 |
| 桌面版 | Windows / macOS / Linux 的 Electron 内置 Node | 只实测了 `ELECTRON_RUN_AS_NODE=1` 跑测试脚本这一半；macOS 与 Alpine/musl 未实测 |

### 3.3 依赖与权限

零依赖：`package.json` 没有 `dependencies` / `devDependencies` / `peerDependencies`，`net-scope-ranges.json` 与 `default-prompt.md` 都是随包静态数据。不需要 root、不需要 Android 权限，只要 `$DSH_HOME` 可写；不发网络请求、不开监听端口、不调用外部程序。

### 3.4 磁盘占用

| 项 | 大小 |
|---|---|
| 发布树全量（含 `test/` `scripts/` `docs/`，不含 `.git/`、`dist/`） | ≈ 0.74 MiB |
| 按 `files` 白名单进包的部分 | ≈ 0.63 MiB |
| 发布 tgz（压缩后） | ≈ 218 KiB |
| `du` 报告的块占用 | 约 958 KiB（小块文件多，块对齐放大了数字） |
| 其中 `assets/net-scope-ranges.json` | 250,787 字节 ≈ 245 KB（v4 6354 段 / v6 2792 段） |
| 其中 `assets/default-prompt.md` | 25,240 字节 ≈ 24.7 KB |
| 首次运行落盘 | `prompt-inject.md` ≈ 26 KB（默认正文 + 分类层），`state.json` 数 KB，`revert` 每次备份一份 |

## 4. 支持设备与宿主矩阵

| 宿主形态 | 注入 | 命令 | 工具 | 工具卡 | 原生状态条/面板 |
|---|---|---|---|---|---|
| 手机本机引擎（dsh-go / DSHBox，Android + PRoot） | ✅ | ✅ | ✅ | ✅ | ✅ |
| SSH 主机 / 中转主机（dsh-go 遥控的远程 DSH） | ✅ | ✅ | ✅ | ✅ | ❌ |
| 桌面版 DSH（Windows / macOS / Linux，Electron 引擎） | ✅ | ✅ | ✅ | ❌ | ❌ |
| 纯 iOS / Android 原生 App（无 PRoot、无 DSH 引擎） | ❌ 不存在该引擎形态 | ❌ | ❌ | ❌ | ❌ |

要点：

- 注入、命令、工具三列在三种 DSH 宿主上都是同一套代码路径——**它们只依赖 `systemPrompt` / `commands` / `tools` 三个核心服务**，任何 0.2.x 引擎都有。
- 工具卡与原生界面走 `ctx.inject(['dshGo'])`；拿不到 `dshGo` 的那段代码整体不执行，其余部分照常工作，所以桌面版装它也不会报错，只是没人渲染那些卡。
- 没有 PRoot 的纯原生 iOS / Android App 里不存在这个引擎形态，不属于本包的支持面。
- **证据等级**：本机只实测了「手机本机引擎」这一行（Android 15 aarch64 + Debian 13 PRoot + DSH 0.2.2-rc.1）；SSH 主机、中转主机、桌面版是按源码路径判定「可用」，未在本机接远端/桌面复测——每行的实测状态见兼容性文档的总表。

每个宿主的部署形态、逐步验证方法与已知坑，见 **[docs/COMPATIBILITY.md](docs/COMPATIBILITY.md)**。

## 5. 安装

三条路，装到同一个地方（profile 的 `node_modules` + `dsh.profile.bundles`），选一条即可。

### 5.1 `dsh plugin add`（推荐）

`dsh plugin` 只是把剩下的参数转给 profile 目录里的 pnpm，所以目录、tgz、npm spec 都能吃：

```bash
# 本地目录（开发用）
dsh plugin --profile web add /绝对路径/dsh-purge-go

# GitHub Release 的 tgz
dsh plugin --profile web add ./dsh-purge-go-0.2.2.tgz

# 自打一份（不改仓库状态，产物在 dist/ 或当前目录）
npm pack

# 验证：组合进去没有
dsh --profile web --dump-config | grep -A2 dsh-purge-go
```

**装完重启一次宿主引擎**：profile 的 bundle 列表在启动时组合，运行中的进程不会热加载新 bundle。重启之后插件随宿主加载，此后改正文、开关、分类都是即时生效（每次组装都重读 state 与文件），不用再重启。

### 5.2 自解压安装器

Android / PRoot 上没有 pnpm 时用这个。它把包解到 `$DSH_HOME/dsh-go/plugins/dsh-purge-go`，改写 profile 的 `package.json`（`dependencies["dsh-purge-go"] = "link:../../dsh-go/plugins/dsh-purge-go"`，并把自己追加进 `dsh.profile.bundles`），在 `node_modules` 里建软链，改之前先备份 profile 配置：

```bash
bash dsh-purge-go-0.2.2-selfinstall.sh --check   # 只体检：profile 登记 / 安装目录 / import / 红队资产
bash dsh-purge-go-0.2.2-selfinstall.sh           # 安装
bash dsh-purge-go-0.2.2-selfinstall.sh --uninstall

DSH_HOME=/custom/path DSH_PROFILE=web bash dsh-purge-go-0.2.2-selfinstall.sh   # 指定 home / profile
```

已实测：真装 + profile 登记 + `node_modules` 软链 + `import` 全通过。`npm pack` 产出的 `package/` 顶层目录由安装器自己改名，用户不用管；旧版本会挪到 `dsh-purge-go.bak-<时间戳>`，profile 配置备份在 `$DSH_HOME/backup-before-dsh-purge-go-<时间戳>/`。

### 5.3 GitHub Release 的 tgz

```bash
sha256sum -c sha256sums.txt
dsh plugin --profile web add ./dsh-purge-go-0.2.2.tgz
```

### 5.4 升级 / 卸载 / 回滚

| 动作 | 命令 |
|---|---|
| 升级 | `dsh plugin --profile web add ./dsh-purge-go-0.2.3.tgz`（覆盖安装，状态与正文保留） |
| 卸载（pnpm 路径） | `dsh plugin --profile web remove dsh-purge-go` |
| 卸载（安装器路径） | `bash dsh-purge-go-0.2.2-selfinstall.sh --uninstall` |
| 回滚 | 重新装上一版：`dsh plugin --profile web add ./dsh-purge-go-0.2.2.tgz` |

卸载不会删你的数据：`$DSH_HOME/dsh-purge/`（状态、变体、备份、分类覆盖层）与 `prompt-inject.md` 都留着。安装器的 `--uninstall` 会顺带清掉它同步过去的 `redteam/skills/*.md` 与 `.agent-presets/redteam/`。要彻底清干净：

```bash
rm -rf "$DSH_HOME/dsh-purge" "$DSH_HOME/redteam" "$DSH_HOME/.agent-presets/redteam"
rm -f "$DSH_HOME/prompt-inject.md" "$DSH_HOME/net-scope-allow.txt"
```

## 6. 快速开始

装完重启宿主，在会话里：

```text
/purge status          # 开关、正文体积与来源、变体、压掉了哪些段、DSH_HOME
/purge apply           # 开注入 + 确保 prompt-inject.md 有正文
```

首次加载会自动 seed 正文并写 `state.json`（`enabled: true`），所以多数情况下 `/purge status` 已经能直接看到一份在跑的正文。想换内容：`/purge prompt set /绝对路径/我的正文.md`。

## 7. 命令

全部挂在 `/purge` 下，空参数等价 `status`。

| 用法 | 说明 |
|---|---|
| `/purge status` | 开关、字节数/字数、来源、变体、主文件、按模型指定、上次压掉的官方段 |
| `/purge on` · `/purge off` | 只切开关（`enable` / `disable` 同义） |
| `/purge apply [force]` | 开注入并确保正文落盘；`force` 覆盖重写默认正文 |
| `/purge revert` | 关注入，并把 `prompt-inject.md` 备份到 `dsh-purge/backups/`（`rollback` 同义） |
| `/purge seed [force]` | 落盘包内默认正文（已存在时不覆盖，`force` 强制） |
| `/purge prompt show` | 看当前真正会注入的正文前 1200 字 |
| `/purge prompt set <文件>` | 用该文件全文替换 `prompt-inject.md`（上限 200,000 字符） |
| `/purge prompt reset` | 恢复包内默认正文 |
| `/purge variant list` | 列出 `dsh-purge/prompts/*.md`，`*` 标当前 |
| `/purge variant use <名>` | 整体切到某个变体 |
| `/purge variant default` | 回默认变体 |
| `/purge variant map <模型> <名\|->` | 给某个模型单独指定变体；`-` 取消 |
| `/purge cat list` | 题材分类：编号 · id · 别名 → 路由，关掉的打 `x` |
| `/purge cat show <id>` | 看一条分类的规则正文与来源 |
| `/purge cat on\|off <id>` | 单条开关 |
| `/purge cat add <id> <别名,别名> <规则正文> [\| 路由]` | 加一条，编号自动接在后面 |
| `/purge cat rm <id>` | 删一条（出厂项写墓碑，`reset` 可复原；`del` / `remove` 同义） |
| `/purge cat reset` | 丢掉覆盖层，回出厂分类 |
| `/purge cat bake` | 把当前分类真正写进 `prompt-inject.md` |
| `/purge home` | 打印 `DSH_HOME` 与相关路径 |
| `/purge help` | 用法（`?` 同义） |

`cat` 也可以用 `category` / `categories`。

## 8. 工具

| 工具 | 参数 | 用途 |
|---|---|---|
| `purge_status` | — | 状态快照：开关、字节/字数、来源、变体、模型、变体数、主文件、压掉的段、home |
| `purge_apply` | `force?: boolean` | 开注入 + 确保正文落盘，返回是否新写、当前字节 |
| `purge_revert` | — | 关注入 + 把正文备份到 `dsh-purge/backups/` |
| `purge_prompt` | `action`: `show`/`set`/`reset`/`use`/`default`/`map`；`text` / `name` / `model` | 读写正文与变体；`show` 附 1200 字预览 |
| `purge_category` | `action`: `list`/`add`/`rm`/`on`/`off`/`reset`/`bake`；`id` / `labels` / `rule` / `route` | 加删题材分类，改完即生效 |

五个工具都通过 `output.presentationMeta` 挂一张 `{"dsh-go/view": …}` 工具卡：根组件 `stack`、预算 8 KiB、节点数 ≤40、只读（卡上没有可点按的动作）。没有 `dshGo` 桥接的宿主不会渲染它，但调用不受影响。

## 9. 配置

`apply(ctx, config)` 的键（`DEFAULTS` 全量，共 22 个）：

| 键 | 默认 | 说明 |
|---|---|---|
| `enabled` | `true` | 首次启动写进 `state.json` 的初值；之后以 state 为准 |
| `dropConflicts` | `true` | 压掉 `harness:identity` / `deployment:persona-prefix` / `-suffix` |
| `capabilityProtocol` | `true` | 注入 `dshpg-capabilities` 段 |
| `postPrompt` | `''` | 额外后置段正文；非空才注册 |
| `postPromptOrder` | `5100` | 后置段 order，必须大于 `-100000` |
| `stripMnemon` | `true` | 滤掉 mnemon 的记忆消息 |
| `seedPrompt` | `true` | 启动时确保 `prompt-inject.md` 存在 |
| `injectSection` | `true` | 关掉后完全不注册注入段（只用命令/工具/面板） |
| `maxPromptBytes` | `262144` | 注入正文上限，超出截断（面板与状态会提示） |
| `tools` | `true` | 关掉工具注册 |
| `commands` | `true` | 关掉 `/purge` |
| `native` | `true` | 关掉 dsh-go 原生界面 |
| `regionGate` | `true` | 地区门（`agent/pre-step`） |
| `recoverRefusal` | `true` | 拒答续跑（`agent/status`） |
| `recoverMax` | `2` | 每会话续跑次数上限 |
| `recoverCooldownMs` | `60000` | 两次续跑之间的冷却 |
| `recoverText` | `''` | 空表示用内置那句续跑话术 |
| `objectiveCard` | `true` | 目标卡段 `dshpg-objective` |
| `redteamAssets` | `true` | 启动时同步随包技能与预设 |
| `categories` | `true` | 题材分类扩展层参与组装 |
| `verbose` | `false` | 打开后 `console.warn` 打注册失败原因 |
| `home` | `''` | 覆盖 `DSH_HOME`（测试与多实例用） |

改配置走 profile 的补丁层（**`config` 整块替换，不做深合并**；没写的键回落到 `DEFAULTS`，而不是上一层的值）：

```yaml
# $DSH_HOME/profiles/web/cordis.patch.yml
- id: dsh-purge-go
  config:
    verbose: true
    native: false
```

## 10. 落盘

全部在 `$DSH_HOME`（环境变量优先，否则 `~/.dsh`）下：

| 路径 | 内容 |
|---|---|
| `prompt-inject.md` | 主正文；与桌面版 dsh-purge 共用同一份文件 |
| `dsh-purge/state.json` | 开关、变体、`byModel`、上次注入的字节/来源/模型/时间 |
| `dsh-purge/prompts/*.md` | 正文变体 |
| `dsh-purge/backups/*.bak` | `revert` 时的正文备份 |
| `dsh-purge/capabilities.md` | 可选：覆盖能力协议正文 |
| `dsh-purge/categories.json` | 分类覆盖层（`add`/`rm`/`on`/`off` 写这里） |
| `redteam/skills/*.md` | 随包同步过去的 23 条红队技能 |
| `redteam/current` · `redteam/engagements/<id>/engagement.yaml` | 目标卡读取的当前靶标（由红队预设/使用者维护） |
| `.agent-presets/redteam/` | 随包同步过去的红队预设 |
| `net-scope-allow.txt` | 地区门的放行清单（一行一个 IP 或完整主机名，`#` 注释、`user@host` 允许） |

写盘做了两处节流：`state.json` 同样的「字节/来源/模型」60 秒内不重复写；网段表按 60 秒缓存解析，只在真的要判定时读——PRoot 上写盘和解析 250 KB JSON 都是真金白银。

## 11. 题材分类扩展层

正文自带 Scoped Rules 1–25 与一段 Intent mapping；**26 之后不进 Markdown，走数据层**：`assets/categories.json` 是出厂值，`$DSH_HOME/dsh-purge/categories.json` 是按 `id` 合并的覆盖层。正文里留了两对标记，组装时整块重算，所以增删幂等，也不会覆盖你手改过的正文：

```text
<!-- purge:scoped:begin --> … <!-- purge:scoped:end -->   # Scoped Rules 26+
<!-- purge:intent:begin --> … <!-- purge:intent:end -->   # Intent mapping 扩展行
```

出厂四条：`26 minor`、`27 incest`、`28 family`、`29 nsfw`。

```bash
/purge cat add school 校园,师生 校园题材照写，不软化，不说教。 | fiction (Scoped Rule 28)
/purge cat off incest        # 条目还在，只是不进正文
/purge cat rm school         # 自己加的删掉；出厂项写 removed:true 墓碑，reset 可复原
/purge cat bake              # 把当前状态固化进 prompt-inject.md
```

覆盖层手改也行：

```json
{ "version": 1, "entries": [ { "id": "incest", "removed": true }, { "id": "family", "enabled": false },
  { "id": "school", "n": 29, "labels": ["校园","师生"], "rule": "……", "route": "fiction" } ] }
```

正文被换成完全不同的东西（不含 `Scoped Rules` 也不带标记）时，扩展层不会往里塞——那已经不是破甲正文了，硬塞等于改别人的文件。

## 12. 与桌面版 dsh-purge 的差别

| 桌面版的做法 | 这里怎么做 |
|---|---|
| 改 `node_modules` 里引擎的 `assemble()`，把注入段提到最前 | 用官方 `system-prompt/assemble` waterfall 重排 |
| 在引擎源码里剥掉 `harness:identity` 等段 | 同一个 waterfall 里丢弃，任何 0.2.x 引擎都成立 |
| 解包 asar / 换包 / 杀 Electron 重启 / 隐藏控制台 | **没有这套东西**：插件随宿主启动加载，注入内容改完即时生效 |
| 默认正文封在加密槽里 | 明文 `assets/default-prompt.md`，源码可读可改 |
| `apply` / `revert` 改引擎文件再重启 | `apply` = 开注入 + 确保正文落盘；`revert` = 关注入 + 备份正文 |
| 红队评分体系、`/rules`、`/skills`、`/rewind`、自动更新、解除安装 | **不在本包**：这些是桌面版的外壳设施 |
| 专门的 net-scope 模块与红队工具子插件 | 内核版地区门（随包网段表）+ 目标卡；工具由本插件自己注册 |

**共存策略**：两个都装会撞 `/purge` 命令与 5 个工具名。本包对注册失败只 `warn` 不抛错，所以加载不会垮，但两个插件的状态文件互相覆盖（都写 `<DSH_HOME>/dsh-purge/state.json` 与 `prompt-inject.md`）。**建议一个宿主只装一个**；真要在同一台机器上对比，用不同 `DSH_HOME`（或本包的 `home` 配置）隔开。

## 13. 已知限制

- **工具卡只读**：dsh-go 第 0 期不支持工具卡上的点按，改状态请用面板或命令。
- **面板刷新依赖桥接 `refresh()`**：老桥接没有 `refresh` 时，状态条走插件自己保存的 push 回调兜底，面板要等下一次打开。
- **不写 `turn.note`**：`grep -r 'turn.note' lib/` 为空，本包不占用宿主的轮次提示位，留给别的插件。
- **地区门只做离线判定**：域名后缀与字面 IP 用随包网段表与后缀表判，不查 DNS、不发请求；表里没有的域名不会被判进来，需要放行就写 `net-scope-allow.txt`。
- **拒答续跑的账本在内存里**：进程重启后计数清零，这是为了 PRoot 上少写盘。
- **不含桌面版的评分/规则/rewind/更新器**：要那些请用桌面版 dsh-purge。
- **同机双装会撞名**：见 §12，只警告，不保证行为。
- **未实测**：macOS、Alpine/musl、Windows arm64、纯 Termux 原生 Node（非 PRoot）、DSH 0.3.x。

## 14. 开发与自测

```bash
node --test test/*.test.js        # 56 例：正文/状态/组装/视图/加载/分类/拒答/红队
node scripts/engine-smoke.mjs     # 真引擎：段顺序、正文体积、工具 schema、工具卡、关闭放行
node scripts/kit-check.mjs        # dsh-go 的 plugin-kit 校验（清单 / 贡献 / 视图）
```

`kit-check.mjs` 需要 dsh-go 仓库的 `sdk/plugin-kit`（`DSH_GO_KIT` 指过去，或放到 `../_ref/kit`），拿不到会直接打印怎么取。

环境变量：

| 变量 | 用在哪 | 作用 |
|---|---|---|
| `DSH_HOME` | 运行时 / 安装器 | 落盘根；不设则 `~/.dsh` |
| `DSH_PROFILE` | 安装器 | 目标 profile，默认 `web` |
| `DSH_ENGINE_DIR` | `engine-smoke.mjs` | 引擎根目录（需含 `node_modules/@deepseek-ai/cordis`） |
| `DSH_GO_KIT` | `kit-check.mjs` | plugin-kit 目录 |
| `REDTEAM_PRESET_REFRESH` | 运行时 | 设 `1` 时覆盖已有的红队预设 |
| `DSH_REDTEAM_HOME` | 目标卡 | 覆盖 `redteam/` 根目录 |

本机实测（DSH `0.2.2-rc.1`，Node `v24.19.0`，`node --test` 56/56 通过）：

```text
段顺序： dsh-purge > dshpg-capabilities > dshpg-objective > agent:instructions
首段： dsh-purge | 正文 26794 字节
工具： purge_status, purge_apply, purge_revert, purge_prompt, purge_category
工具卡： 390 字节，根组件 stack | 标题 破甲状态
```

「关闭后」自有段仍在列表上但正文是空串——引擎渲染时跳过空段，实际 prompt 与原生一致（烟测里断言 `harness:identity` 会回来）。

Windows 上没有 `node` 在 PATH 时，用桌面版自带的 Node：

```bash
ELECTRON_RUN_AS_NODE=1 "D:/DEEPSEEK/DeepSeek Harness.exe" --test test/*.test.js
ELECTRON_RUN_AS_NODE=1 "D:/DEEPSEEK/DeepSeek Harness.exe" scripts/engine-smoke.mjs
```

## 15. 目录结构

```text
dsh-purge-go/
├── package.json              # dsh.bundle.patch + dshGo 清单 + engines
├── cordis.patch.yml          # 把自己 insert 进插件树
├── assets/
│   ├── default-prompt.md     # 24.7 KB 默认正文（明文，Scoped Rules 1-25 + 两对分类标记）
│   ├── categories.json       # 分类扩展层出厂数据（26-29）
│   └── net-scope-ranges.json # 245 KB 网段表（v4 6354 / v6 2792）
├── lib/                      # 18 个 ESM 模块
│   ├── index.js              # apply()：段、waterfall、mnemon、region、recover、tools/commands/dshGo
│   ├── home.js  state.js     # DSH_HOME/路径/原子写/备份；state.json 读写
│   ├── prompt.js  categories.js  capability.js   # 正文解析与变体；分类扩展层；能力协议段
│   ├── assemble.js  view.js  card.js             # waterfall 重排；快照与视图；工具卡预算
│   ├── ops.js  native.js                         # 开关/回滚/落盘；dshGo strip/panel/on/refresh
│   ├── tools.js  commands.js                     # 5 个工具；/purge
│   └── region.js  region-ip.js  objective.js  recover.js  redteam.js  # 地区门/IP 判定/目标卡/拒答续跑/技能同步
├── skills/redteam/*.md       # 23 条红队技能
├── presets/redteam/          # agent.cordis.yml + preset.yml + anchor-gate.mjs
├── scripts/                  # engine-smoke.mjs · kit-check.mjs · build-selfinstall.sh · selfinstall-head.sh
├── test/*.test.js            # node:test，56 例
├── docs/COMPATIBILITY.md     # 兼容性与设备支持详解
└── dist/                     # 发布产物（不入 tgz）
```

进 tgz 的只有 `files` 白名单（`lib` / `assets` / `skills` / `presets` / `cordis.patch.yml` / `README.md` / `package.json`）+ npm 自动带的 `LICENSE`；`test/`、`scripts/`、`docs/`、`dist/` 不进包。

## 16. 发布与版本

- **tag**：`v0.2.2`（`v` + `package.json` 的 `version`）。
- **GitHub Release 资产**（三个，一起传）：
  - `dsh-purge-go-0.2.2.tgz` — `npm pack` 同款包，喂给 `dsh plugin add` / `npm i`；
  - `dsh-purge-go-0.2.2-selfinstall.sh` — 自解压安装器（内含 base64 payload，Android / PRoot / 任意 Linux 可跑）；
  - `sha256sums.txt` — 上面两个文件的校验和，`sha256sum -c sha256sums.txt`。
- **生成入口**：`bash scripts/build-selfinstall.sh`（`OUT=` 换输出目录）。它按 `package.json` 的版本号产出上述三个文件，并自带真装自检：payload 可解、`package/lib/index.js` 与 `package/LICENSE` 在、`--check` 通过、实装后 profile 登记与 `node_modules` 软链都对。asar / PowerShell / 单独的打包机都不需要。
- **版本语义**：遵守 SemVer。`0.x` 期间次要版本可能带行为变更（比如段的默认开关），补丁版本只修 bug；破坏性变更写进 CHANGELOG 的 `BREAKING`。`engines` 的 `dsh` 范围是硬边界，超出范围不做承诺。
- **CHANGELOG** 由维护者维护，跟 tag 一起发。

## 17. 许可与合规

- 本包以 **AGPL-3.0** 发布，全文见 [LICENSE](LICENSE)。
- **网络分发也须提供源码**（AGPL §13）：把它跑成对外服务的人，需要向使用者提供对应源码（含你对本包的修改）。
- **修改后再分发须同许可**：衍生作品同样以 AGPL-3.0 分发，并保留版权与许可声明。
- **无担保**：按「现状」提供，不附带任何明示或默示担保。
- **默认正文来源**：`assets/default-prompt.md` 的正文源自上游 dsh-purge，随包原样保留。若上游对其再分发有异议，删掉/替换 `$DSH_HOME/prompt-inject.md`（或改 `assets/default-prompt.md` 后重打包）即可，代码部分不依赖这段正文的具体内容。
- **合规与授权责任在部署方**：本包只提供提示词组织与工具接线，不判断你拿它做什么。用它做什么、对哪些目标做、是否需要授权，由部署与使用它的人负责。

## 18. 免责声明 · 贡献 · 安全披露 · 致谢

**免责声明**：本软件按「现状」提供，不附带任何明示或默示担保，包括但不限于适销性、特定用途适用性与非侵权担保。作者与贡献者不对因使用本软件产生的任何直接或间接损失负责。使用前请自行确认所在环境的法律与授权要求。

**贡献**：欢迎 issue 与 PR。提交前请保证 `node --test test/*.test.js` 全绿；改到提示词段、工具 schema、dsh-go 清单时，附上 `node scripts/engine-smoke.mjs` 的输出。新增行为请补测试，不要只改文档。

**安全披露**：安全问题请发 `security@example.com`（**占位符，发布前替换**），不要开公开 issue。请附版本号、宿主形态、复现步骤与相关日志；72 小时内会给首次回复。

**致谢**：

- [dsh-go](https://github.com/ccch1mneyyy/dsh-Go) —— 手机端的宿主与原生插件协议（状态条 / 面板 / 工具卡 / 清单）；
- **dsh-purge** —— 桌面版的注入内核与默认正文，本包是它的独立重写；
- **DeepSeek Harness** —— 引擎与官方扩展点，没有 `systemPrompt` / `system-prompt/assemble` / `agent/*` 这些口子，这份包只能继续改引擎文件。

## 发布前替换清单

| 占位符 | 出现位置 | 替换成 |
|---|---|---|
| `OWNER` | 文首 CI 徽章（2 处 URL） | GitHub 用户名或组织名 |
| `security@example.com` | §18 安全披露 | 真实安全邮箱 |
| 仓库 slug `OWNER/dsh-purge-go` | 徽章、Release 链接 | 真实仓库地址 |

一次性替换（README / CHANGELOG / package.json 里的 `OWNER` 全部换掉）：

```bash
bash scripts/set-repo-owner.sh <你的GitHub用户名>          # 先 --dry-run 看会改哪些行
```

顺带确认：`.github/workflows/ci.yml` 已推送（否则徽章 404）；`LICENSE` 为 AGPL-3.0 全文；`CHANGELOG.md` 的 `0.2.2` 段与本次 tag 一致；Release 三个资产齐全且 `sha256sums.txt` 与它们对得上。

## 出自

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
