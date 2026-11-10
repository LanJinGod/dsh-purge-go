# 兼容性与设备支持

本文件展开 [README](../README.md) §4 的支持矩阵：每种宿主怎么部署、怎么一步步验证、以及实测踩到的坑。

**证据分级**（全文统一）：

- **实测** —— 在本机样本上跑过，命令与输出可复现；
- **理论可用，未实测** —— 代码路径成立（只依赖核心服务，或缺 `dshGo` 时整段跳过），但没有在真机上跑过，不给结论；
- **不支持** —— 该形态里不存在本插件赖以工作的引擎服务。

---

## 0. 结论速览

- 唯一硬要求：**宿主是 DSH 引擎**，版本 `>=0.2.1-rc.2 <0.3.0-0`，Node `>=22.19`。
- 注入、`/purge`、5 个工具在三种 DSH 宿主上走**同一条代码路径**，只依赖 `systemPrompt` / `commands` / `tools`。
- 工具卡与原生状态条/面板走 `dshGo` 桥接，**只有 dsh-go 系宿主有**；没有桥接时那段代码整体不执行，其余照常。
- 没有 PRoot 的纯原生 iOS / Android App 里没有这个引擎形态；裸 Node 环境里也没有 `systemPrompt` 服务。
- 实测样本只有一套：Android 15 + Debian 13 PRoot + aarch64 + DSH `0.2.2-rc.1` + Node `v24.19.0`。其它平台见 §10 未验证清单。

---

## 1. 总表

| # | 宿主形态 | 引擎来源 | 注入 | 命令 | 工具 | 工具卡 | 原生面板 | 状态 |
|---|---|---|---|---|---|---|---|---|
| A | 手机本机引擎（dsh-go / DSHBox，Android + PRoot） | PRoot 容器内的 DSH | ✅ | ✅ | ✅ | ✅ | ✅ | **实测**（注入/工具经 engine-smoke 与 `/purge`；原生界面在 App 内可见） |
| B | SSH 主机（dsh-go 遥控的远程 DSH） | 远端主机上的 DSH | ✅ | ✅ | ✅ | ✅ | ❌ | 与 A 同一代码路径，**理论可用，未实测**（本机没接远端）；工具卡由 App 侧渲染，取决于桥接版本 |
| B2 | 中转主机（经中转节点连到的远端 DSH） | 同 B | ✅ | ✅ | ✅ | ✅ | ❌ | 与 B 同一代码路径，**理论可用，未实测** |
| C | 桌面版 DSH（Windows / macOS / Linux，Electron） | Electron 内置 Node | ✅ | ✅ | ✅ | ❌ | ❌ | 注入/命令/工具 **理论可用，未实测**（Electron 侧只实测了 `ELECTRON_RUN_AS_NODE` 跑脚本） |
| D | 纯 iOS / Android 原生 App（无 PRoot） | 无 DSH 引擎 | ❌ | ❌ | ❌ | ❌ | ❌ | **不支持** |
| E | 裸 Node（`node lib/index.js`，无 DSH） | 无 | ❌ | ❌ | ❌ | ❌ | ❌ | **不支持**（能 import，但没有 `systemPrompt` 服务，只写文件不注入） |

> 工具卡一列说的是「卡片数据有没有宿主来渲染」：卡片由本插件生成（`output.presentationMeta` → `{"dsh-go/view": …}`），渲染方是 dsh-go App。桌面版没有这个渲染方，卡片只是没人读的元数据。

### 1.1 实测到底测了什么

| 检查 | 命令 | 结果 |
|---|---|---|
| 单元测试 | `node --test test/*.test.js` | 56 例全通过 |
| 真引擎烟测 | `DSH_ENGINE_DIR=/opt/dshapp/runtime node scripts/engine-smoke.mjs` | 段顺序、正文 26794 字节、5 个工具、工具卡 390 字节、关闭放行 全部 OK |
| 包内容/装载 | 自解压安装器 `--check` 与真装 | profile 登记 + `node_modules` 软链 + `import` 全通过 |
| 会话内 | `/purge status`、`/purge home` | 开关、来源、体积、路径均正确 |

烟测的实际输出（DSH `0.2.2-rc.1`）：

```text
宿主自带段： harness:identity > deployment:persona-prefix > deployment:persona-suffix
段顺序： dsh-purge > dshpg-capabilities > dshpg-objective > agent:instructions
首段： dsh-purge | 正文 26794 字节
工具： purge_status, purge_apply, purge_revert, purge_prompt, purge_category
工具卡： 390 字节，根组件 stack | 标题 破甲状态
关闭后： dsh-purge > dshpg-capabilities > dshpg-objective > harness:identity > deployment:persona-prefix > agent:instructions > deployment:persona-suffix
```

「关闭后」自有段仍在列表里但正文是空串，引擎渲染时跳过空段——实际 prompt 与原生一致，这一点烟测里是断言过的（关闭后 `harness:identity` 必须回来）。

---

## 2. 设备与架构

| 项 | 实测值 | 说明 |
|---|---|---|
| 宿主系统 | Android 15 | dsh-go / DSHBox 跑在手机上 |
| 内核 | 6.6.118 | `uname -r` |
| 容器 | Debian GNU/Linux 13 (trixie) PRoot | 引擎与 Node 都在容器里 |
| 架构 | aarch64 | 实测样本 |
| Node | v24.19.0 | 满足 `>=22.19` |
| DSH | 0.2.2-rc.1 | 满足 `>=0.2.1-rc.2 <0.3.0-0` |
| 本包 | 0.2.2 | — |

代码里没有原生二进制、`.node`、按架构分支的逻辑，所以理论上与架构无关；但**只实测了 aarch64**。x86_64 / arm64 桌面、macOS、Windows 原生、Alpine/musl、纯 Termux（非 PRoot）都未实测，不给结论。

---

## 3. 宿主 A：手机本机引擎（dsh-go / DSHBox）

### 3.1 部署形态

引擎、Node、本插件全在同一台手机的 PRoot 容器里。App 是外壳，引擎是 App 拉起的 Node 进程，插件是引擎里的一个 Cordis 插件。

- `DSH_HOME` 默认 `~/.dsh`（容器内 root 用户即 `/root/.dsh`）；App 若改了 home，以 `/purge home` 打印的为准。
- 落盘都在这一个目录下：`prompt-inject.md`、`dsh-purge/`、`redteam/`、`.agent-presets/`、`profiles/<profile>/`。

### 3.2 装法

在 App 的终端里（或 `adb shell` 进容器）：

```bash
node -v            # 必须 >= 22.19
dsh --version      # 必须落在 0.2.1-rc.2 .. 0.3.0-0

# 方式一：pnpm 路径（有 pnpm 时）
dsh plugin --profile web add /root/dsh-purge-go

# 方式二：自解压安装器（没有 pnpm 时）
bash dsh-purge-go-0.2.2-selfinstall.sh --check
bash dsh-purge-go-0.2.2-selfinstall.sh
```

**装完必须完全重启引擎**（退出 App 后重开，或重启 App 内的引擎进程）。只刷新会话页不够：profile 的 `dsh.profile.bundles` 是启动时组合的。

### 3.3 逐步验证

```bash
# 1. profile 里登记了没有
dsh --profile web --dump-config | grep -A2 dsh-purge-go

# 2. 包能否被 import（不启动引擎也能查）
node -e 'const {pathToFileURL}=require("node:url");import(pathToFileURL(process.argv[1]+"/lib/index.js").href).then(m=>console.log(Object.keys(m).sort().join(",")))' "$PWD"

# 3. 安装器视角的体检
bash dsh-purge-go-0.2.2-selfinstall.sh --check
```

重启后在会话里：

```text
/purge home      # 先确认 DSH_HOME 是哪个
/purge status     # 破甲：开 · N 字节 / M 字 · 来源 … · 变体 default
```

状态条应显示 `破甲 开 · 26.x KB`（≤24 字），破甲面板里能看到正文体积、来源、变体、压掉的段、地区表 `v4 6354 / v6 2792`、红队资产 `23 技能 · 预设已装`。

### 3.4 已知坑

| 坑 | 现象 | 处理 |
|---|---|---|
| PRoot 写盘贵 | 每步组装都写 `state.json` 会明显卡顿、耗电 | 已内置节流：同样的「字节/来源/模型」60 秒内不重复写；网段表 60 秒缓存。不要把它调小 |
| `revert` 备份堆积 | `dsh-purge/backups/*.bak` 越来越大 | 每次 `revert` 一份，定期清理旧文件；备份的是 `prompt-inject.md` |
| DSH_HOME 在 `/sdcard` | FUSE 层慢、还可能被系统回收权限 | 把 home 放在容器 rootfs 内（默认 `~/.dsh`）；要备份就把 `dsh-purge/` 与 `prompt-inject.md` 拷出来 |
| 清了容器 | 整个 rootfs（含 home）一起没 | 升级镜像前先导出 `prompt-inject.md` + `dsh-purge/` |
| Node 太老 | 插件不出现，或启动时报加载错误 | `node -v` 先看；用容器里装 22.19+，或改用引擎自带的那个 node |
| 端口/进程残留 | 重启后旧引擎还在跑，看到两套状态 | 完全退出 App 再重开，别只关页面 |

---

## 4. 宿主 B：SSH 主机 / 中转主机（遥控的远程 DSH）

### 4.1 部署形态

dsh-go 只是遥控端，引擎跑在远端 Linux 主机上。本插件装**在远端**，不是装在手机上。

- 有 `dshGo` 桥接（遥控通道）→ 工具卡能在 App 里看到；
- 没有桥接 → 那段代码不执行，注入/命令/工具照常。

### 4.2 装法

```bash
ssh HOST 'node -v && dsh --version'
ssh HOST 'dsh plugin --profile web add ~/dsh-purge-go'          # 目录先 scp 上去
# 或者
scp dsh-purge-go-0.2.2.tgz HOST:/tmp/
ssh HOST 'dsh plugin --profile web add /tmp/dsh-purge-go-0.2.2.tgz'
ssh HOST 'bash /tmp/dsh-purge-go-0.2.2-selfinstall.sh --check'  # 安装器路径
```

远端装完要**重启远端的引擎进程**，遥控端重新连一次会话。

### 4.3 逐步验证

```bash
ssh HOST 'dsh --profile web --dump-config | grep -A2 dsh-purge-go'
ssh HOST 'ls -l ~/.dsh/prompt-inject.md ~/.dsh/dsh-purge/state.json'
```

```text
/purge status      # 遥控会话里能返回状态即说明命令注册成功
/purge home        # 确认远端的 DSH_HOME
```

工具侧验一句：让会话调用 `purge_status`，或直接看面板里有没有 `purge` 面板（有桥接才有）。

### 4.4 已知坑

| 坑 | 现象 | 处理 |
|---|---|---|
| 装错端 | 手机上装了、远端没装，`/purge` 不认识 | 插件必须装在被遥控的引擎那一侧 |
| 两个 home | 远端 `DSH_HOME` 与手机不同，正文/状态各自一份 | 用 `/purge home` 确认；要统一就手工拷 `prompt-inject.md` 与 `dsh-purge/` |
| 版本不一致 | 远端 DSH 0.3.x 或 0.2.0，`engines.dsh` 出界 | 先 `dsh --version`；出界不承诺能用 |
| 同机还装着桌面版 | `/purge` 与 5 个工具撞名，只 warn | 见 §9 |
| 无桥接时以为坏了 | 面板/状态条不出现 | 这是预期：遥控端没有 `dshGo` 就没有原生界面 |

---

## 5. 宿主 C：桌面版 DSH（Windows / macOS / Linux）

### 5.1 部署形态

Electron 里的 Node 就是引擎的 Node。插件装进 `desktop` 这个保留 profile，**没有** `dshGo` 桥接，所以没有状态条、面板、工具卡。

### 5.2 装法

```bash
# 桌面版必须先启动过一次，让 profile 初始化；然后完全退出桌面版
dsh plugin --profile desktop add ./dsh-purge-go-0.2.2.tgz

# Windows（PowerShell / cmd）
dsh plugin --profile desktop add D:\path\dsh-purge-go
```

`--profile desktop` 是保留名：profile 目录不存在时命令会让你先开一次桌面版（引擎自己的保护逻辑），照做即可。

### 5.3 逐步验证

```bash
dsh --profile desktop --dump-config | grep -A2 dsh-purge-go
```

桌面版没有 `/purge`？—— 有，命令是引擎层注册的，在桌面版的会话里同样可输。验注入看会话行为与 `$DSH_HOME/prompt-inject.md` 的写入时间。

没有 `node` 在 PATH 时，用桌面版自带的 Node 跑脚本：

```bash
ELECTRON_RUN_AS_NODE=1 "D:/DEEPSEEK/DeepSeek Harness.exe" --test test/*.test.js
ELECTRON_RUN_AS_NODE=1 "D:/DEEPSEEK/DeepSeek Harness.exe" scripts/engine-smoke.mjs
```

### 5.4 已知坑

| 坑 | 现象 | 处理 |
|---|---|---|
| 没有 `node` 命令 | `node --test` 报找不到 | 用上面的 `ELECTRON_RUN_AS_NODE=1` 形式 |
| 桌面版占着 profile | `dsh plugin --profile desktop` 报错要求先退出 | 完全退出桌面版再装 |
| 以为有面板 | 找不到「破甲」面板 | 桌面版无 `dshGo`，只能用 `/purge` 与工具 |
| 无工具卡 | 工具调用正常但没卡片 | 同上，卡片没人渲染 |
| 与桌面版 dsh-purge 同装 | 撞名 + 状态互覆盖 | 见 §9 |
| 路径大小写 | Windows 下 `ELECTRON_RUN_AS_NODE` 的 exe 路径含空格 | 路径整体加引号 |

---

## 6. 不支持的形态

| 形态 | 为什么 | 说明 |
|---|---|---|
| 纯 iOS / Android 原生 App（无 PRoot、无 DSH 引擎） | 没有 DSH 引擎，没有 `systemPrompt` / `tools` / `commands` 服务 | 不存在这个部署形态，不写进支持面 |
| 裸 Node（`node lib/index.js`、自建 HTTP 服务） | `inject = ['systemPrompt']` 等不到服务 | 能 import、能写文件，但不会注入任何提示词 |
| 其它客户端 / 其它 Agent 框架 | 扩展点是 Cordis / DSH 专有的 | 不适用 |

---

## 7. 环境变量与路径

| 名字 | 类型 | 作用 | 默认 |
|---|---|---|---|
| `DSH_HOME` | 环境变量 | 落盘根 | `~/.dsh` |
| `DSH_PROFILE` | 环境变量（安装器） | 目标 profile | `web` |
| `DSH_ENGINE_DIR` | 环境变量（`engine-smoke.mjs`） | 引擎根目录，需含 `node_modules/@deepseek-ai/cordis` | 自动探测 |
| `DSH_GO_KIT` | 环境变量（`kit-check.mjs`） | dsh-go 的 `sdk/plugin-kit` 目录 | `../_ref/kit` 等 |
| `REDTEAM_PRESET_REFRESH` | 环境变量 | 设 `1` 时覆盖已有的红队预设 | 不覆盖 |
| `DSH_REDTEAM_HOME` | 环境变量 | 覆盖目标卡读取的 `redteam/` 根 | `$DSH_HOME/redteam` |
| `home` | 插件配置项 | 覆盖 `DSH_HOME`（多实例/测试） | `''` 表示用环境变量 |

关键路径：

| 路径 | 内容 |
|---|---|
| `$DSH_HOME/profiles/<p>/package.json` | profile 清单：`dependencies` + `dsh.profile.bundles` |
| `$DSH_HOME/profiles/<p>/node_modules/dsh-purge-go` | pnpm 安装位，或安装器建的软链 |
| `$DSH_HOME/dsh-go/plugins/dsh-purge-go` | 自解压安装器的解包位 |
| `$DSH_HOME/prompt-inject.md` | 注入正文（与桌面版共用同一份） |
| `$DSH_HOME/dsh-purge/state.json` | 开关、变体、`byModel`、上次注入信息 |
| `$DSH_HOME/dsh-purge/prompts/*.md` | 变体 |
| `$DSH_HOME/dsh-purge/backups/*.bak` | `revert` 备份 |
| `$DSH_HOME/dsh-purge/categories.json` | 分类覆盖层 |
| `$DSH_HOME/dsh-purge/capabilities.md` | 覆盖能力协议正文（可选） |
| `$DSH_HOME/net-scope-allow.txt` | 地区门放行清单 |
| `$DSH_HOME/redteam/skills/*.md` | 同步过去的 23 条技能 |
| `$DSH_HOME/.agent-presets/redteam/` | 同步过去的红队预设 |

---

## 8. 故障排查

| 症状 | 可能原因 | 处理 |
|---|---|---|
| 段没排到最前 | 引擎没有 `system-prompt/assemble` 事件（版本太老）；或插件没进插件树 | `dsh --version` 对范围；`dsh --profile <p> --dump-config \| grep -A2 dsh-purge-go`；开 `verbose: true` 看 warn；`/purge status` 看「压掉的段」是否为空 |
| 首段不是 `dsh-purge` | 同上有别的插件用 `order < -100000` 抢位 | 先确认本插件已注册；`/purge status` 的字节数非 0 说明正文在 |
| 正文没落盘 | `seedPrompt: false`；`DSH_HOME` 不可写；安装后没重启 | `/purge home` 看路径；`/purge apply` 手动触发；检查目录权限 |
| 状态一直是「关」 | `state.json` 里 `enabled:false` 优先于配置默认值 | `/purge on` 或改 `state.json` |
| 工具卡不显示 | 宿主没有 `dshGo`（桌面 / 裸 Node）；或旧 dsh-go 不认 `presentationMeta` | 预期行为；卡片不影响工具调用 |
| 工具没注册 | 宿主没有 `tools` 服务；或与桌面版 dsh-purge 撞名 | 看 `verbose` 的 warn；撞名见 §9 |
| `/purge` 不认识 | 命令没注册（无 `commands` 服务）；或撞名被桌面版占了 | `/purge help` 试一次；确认只装了一个 |
| `node: command not found` | PATH 里没有 node（Windows 桌面版常见） | 用 `ELECTRON_RUN_AS_NODE=1 "<exe>"` 跑 |
| Node 版本过低 | `node -v` < 22.19 | 升级 Node，或用引擎自带那个 |
| `node_modules` 软链断 | 安装器建的 `link:../../dsh-go/plugins/dsh-purge-go` 目标被删/被移；`DSH_HOME` 搬家 | 重跑 `--uninstall` 再装；或改用 `dsh plugin add` 重装 |
| 预设挂载失败 | `.agent-presets/redteam/` 里残留了引用桌面版子插件（`dsh-purge/redteam/tools`）的旧文件；或 YAML 写坏 | 删掉该目录，`REDTEAM_PRESET_REFRESH=1` 重启让本包重写 |
| 预设「没生效」 | 默认不覆盖用户已有版本 | 确认目录里有 `preset.yml` + `agent.cordis.yml`；要强制覆盖设 `REDTEAM_PRESET_REFRESH=1` |
| 面板不刷新 | 老桥接没有 `refresh()`，状态条走 push 兜底 | 关掉面板重开；或升级 dsh-go |
| 正文超限被截断 | 超过 `maxPromptBytes`（默认 262144） | 调大配置或精简正文；面板会提示「已截断」 |
| 分类没进正文 | 正文被换成完全不同的内容（不含 `Scoped Rules`/标记） | 设计如此：扩展层只改破甲正文；用 `/purge cat bake` 显式写入 |
| 地区门误拦 | 域名后缀或 IP 落在受限范围 | 把目标写进 `$DSH_HOME/net-scope-allow.txt`（一行一个 IP 或完整主机名） |
| 地区门没拦 | 目标是表里没有的域名，且不是字面 IP | 只做离线判定，不查 DNS；需要覆盖就自己补 allow 清单或关掉 `regionGate` |
| 拒答续跑没触发 | 每会话已达 `recoverMax`；冷却未过；用户已追问 | 看配置；账本在内存，进程重启即清零 |

---

## 9. 与桌面版 dsh-purge 共存 / 迁移

### 9.1 共存

| 冲突点 | 表现 | 建议 |
|---|---|---|
| `/purge` 命令名 | 后注册的失败，只 warn | 一个宿主只装一个 |
| 5 个工具名（`purge_*`） | 同上 | 同上 |
| `$DSH_HOME/dsh-purge/state.json` | 两个插件互相覆盖开关与变体 | 用不同 `DSH_HOME`，或本包的 `home` 配置项隔开 |
| `$DSH_HOME/prompt-inject.md` | 同一份正文，谁写谁赢 | 若要对比，两边各自备份一份再切 |

本包对注册失败只 `warn` 不抛错，所以撞名不会拖垮加载，但**行为不保证**——这是有意为之的取舍：宁可半残也要能启动。

### 9.2 迁移

`prompt-inject.md` 与 `dsh-purge/state.json` 与桌面版是同一份路径、同一份格式，所以：

```bash
# 从旧机器带走
tar czf purge-state.tgz -C "$OLD_DSH_HOME" prompt-inject.md dsh-purge

# 到新机器放回去（先停引擎）
tar xzf purge-state.tgz -C "$NEW_DSH_HOME"
```

注意事项：

- `state.json` 本包只认 `version` / `enabled` / `variant` / `byModel` / `seededAt` / `lastApplyAt` / `lastBytes` / `lastSource` / `lastModel` / `lastAt`；桌面版的私有字段在本包第一次写状态时会丢。迁移前先备份原文件。
- `backups/*.bak` 可以不带，只是历史。
- `categories.json` 是覆盖层，带了就一起生效；不带就用出厂四条。
- `redteam/` 与 `.agent-presets/redteam/` 会在启动时按包内版本同步；已有文件默认保留，要覆盖设 `REDTEAM_PRESET_REFRESH=1`。
- 桌面版的评分/规则/rewind/更新器不在本包，迁移不会带过来，也不需要带。

---

## 10. 未验证清单

以下都没在本机跑过，**不给结论**，只列出来方便排查时先排除：

- macOS（Intel / Apple Silicon）上的桌面版 DSH；
- Windows arm64；
- Alpine / musl 环境；
- 纯 Termux（不经 PRoot）里的 Node；
- DSH `0.3.x` 与 `0.2.0` 及更早版本；
- 真机 SSH 主机与中转主机的完整链路（只按代码路径判断成立）；
- 除 aarch64 之外的架构；
- 与桌面版 dsh-purge 同时安装的组合行为（知道会撞名，没测具体后果）。

上报兼容性问题时带这些信息最省事：`uname -a`、`node -v`、`dsh --version`、`/purge home`、`/purge status`、宿主形态（A/B/C）、以及 `verbose: true` 下的 warn 输出。

---

## 出自

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
