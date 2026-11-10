# 更新日志

本文件记录 dsh-purge-go 的对外版本。格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

> 0.2.x 线中的 0.2.1 未对外发布，本仓库自 0.2.2 起公开；此前版本只存在于内部构建。

## [0.2.2] — 2026-11-10

首个公开版本（AGPL-3.0）。相对内部构建，代码行为不变，补齐了开源发布所需的一切。

### 新增

- 题材分类扩展层：`assets/categories.json` 出厂数据 + `<DSH_HOME>/dsh-purge/categories.json` 覆盖层，把 Scoped Rules 26+ 与意图映射扩展行在组装时拼进正文；对应 `/purge cat *` 命令与 `purge_category` 工具。
- `test/categories.test.js`：分类加载、覆盖层合并、标记区渲染与幂等性。
- 仓库文件：`LICENSE`（AGPL-3.0 全文）、`README.md`、`docs/COMPATIBILITY.md`、`CHANGELOG.md`、`CONTRIBUTING.md`、`SECURITY.md`。
- GitHub Actions：`ci.yml`（Node 22/24，跑 `node --test` 并产出 tgz）与 `release.yml`（`v*` tag 触发，校验 tag 与版本一致、生成 `sha256sums.txt`、发布 Release 资产）。
- `scripts/publish-github.sh`：有 `gh` 走 `gh`，没有就走 `curl` + `GITHUB_TOKEN` 的 REST 流程；`--dry-run` 只打印计划，`--bundle` 产出可离线导入的 git bundle / zip。

### 变更

- `package.json`：解除 `private`，补 `keywords` / `author` / `repository` / `homepage` / `bugs` / `publishConfig`，`files` 白名单加入 `LICENSE`；新增 `pack`、`release:dry`、`publish:github` 脚本。
- `scripts/engine-smoke.mjs`：引擎目录改为自动探测（`$DSH_ENGINE_DIR` → 桌面版安装目录 → PATH 上的 `dsh` → 常见安装位），不再硬编码 `D:/DEEPSEEK/resources/app/dsh`；探测失败会打印全部候选目录。

### 验证

- `node --test test/*.test.js`：56 例通过。
- `DSH_ENGINE_DIR=/opt/dshapp/runtime node scripts/engine-smoke.mjs`（真引擎，Android 15 aarch64 + Debian 13 PRoot，DSH 0.2.2-rc.1）：段顺序 `dsh-purge > dshpg-capabilities > dshpg-objective > agent:instructions`，首段正文 26794 字节，5 个工具全部注册且 schema 通过，工具卡 390 字节。

## [0.2.0] — 2026-11-05

把桌面版 dsh-purge 2.2.1 中「不依赖改引擎」的三块能力按同一份逻辑重写进手机端，依旧零依赖。

### 新增

- 地区门：`agent/pre-step`（`{ global: true, prepend: true }`）判定任务文本里的目标，命中禁止范围就写禁止句并收掉这一轮；字面 IP 走随包网段表 `assets/net-scope-ranges.json`（v4 6354 段 / v6 2792 段），纯离线判定，不发 DNS、不发请求。
- 拒答续跑：`agent/status`（`{ global: true }`）在一轮正常结束、最后一条是逃避句、用户没接话、该拒答还没追过时追一句把任务拉回来；账本按 `turn/end` 的 seq 去重，每会话上限默认 2、冷却默认 60 s，且不因新回合清零。
- 目标卡：提示词段 `dshpg-objective`（order `-99998`），读 `<DSH_HOME>/redteam/current` → `engagements/<id>/engagement.yaml`，把「只打：X / 范围：… / 不打：其它单位」写成系统提示；没有当前靶标时返回空串。
- 红队资产同步：23 条随包技能 → `<DSH_HOME>/redteam/skills/`；预设 → `<DSH_HOME>/.agent-presets/redteam/`（已存在则保留用户那份，`REDTEAM_PRESET_REFRESH=1` 才覆盖）。
- 新文件：`lib/region.js`、`lib/region-ip.js`、`lib/objective.js`、`lib/recover.js`、`lib/redteam.js`、`assets/net-scope-ranges.json`、`skills/redteam/*.md`、`presets/redteam/`、`test/recover.test.js`、`test/redteam.test.js`。

### 说明

- 预设里不再引用桌面版的 `dsh-purge/redteam/tools` 子插件——悬空引用会让整份预设挂载失败。
- 网段表按 60 s 缓存解析；拒答账本只放进程内存，不写盘（PRoot 上每轮写状态文件太贵）。

## [0.1.0] — 2026-11-05

首个可运行版本：把桌面版 dsh-purge 的提示词注入内核抽成独立 DSH 插件。

### 新增

- 注入段 `dsh-purge`（order `-100000`，`interpolate: false`），正文来自 `<DSH_HOME>/prompt-inject.md`，缺失时用包内 25 KB 默认正文。
- 能力协议段 `dshpg-capabilities`（order `-99999`），可用 `<DSH_HOME>/dsh-purge/capabilities.md` 覆盖。
- `system-prompt/assemble` waterfall：把本插件的段重排到最前，默认丢弃 `harness:identity`、`deployment:persona-prefix`、`deployment:persona-suffix`。
- 正文变体：`<DSH_HOME>/dsh-purge/prompts/<名字>.md`，可按模型指定（`state.json` 的 `byModel`）。
- `agent/pre-step` 过滤 mnemon 每一步塞进来的记忆消息（`stripMnemon: false` 可关）。
- `/purge` 命令组、5 个工具、dsh-go 状态条 + 2 个面板 + 1 个加号工具，工具卡预算 8 KiB。

[0.2.2]: https://github.com/LanJinGod/dsh-purge-go/releases/tag/v0.2.2
[0.2.0]: https://github.com/LanJinGod/dsh-purge-go/releases/tag/v0.2.0
[0.1.0]: https://github.com/LanJinGod/dsh-purge-go/releases/tag/v0.1.0
