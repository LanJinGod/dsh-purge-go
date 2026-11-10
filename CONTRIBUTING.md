# 参与贡献

感谢愿意花时间。这个包的定位很窄：**只做 DSH 引擎上「提示词正文注入 + 手机端原生界面」这一件事**，所以合并请求会优先看「是否只用了官方扩展点」「是否仍然零依赖」「是否在手机上不炸」。

## 开发环境

| 需要 | 版本 | 说明 |
|---|---|---|
| Node.js | ≥ 22.19 | 只用到内置模块（`node:fs` / `node:path` / `node:test`），不需要安装任何依赖 |
| DSH 引擎 | `>=0.2.1-rc.2 <0.3.0-0` | `scripts/engine-smoke.mjs` 和真机验证要用；纯跑单元测试可以没有 |
| Bash | 3.2+ | 只有发布/安装脚本需要，`build-selfinstall.sh` 在 Linux / macOS / PRoot 上都能跑 |

克隆后不需要 `npm install`，直接：

```bash
node --test test/*.test.js          # 56 例单元测试，约 2 s
DSH_ENGINE_DIR=/opt/dshapp/runtime node scripts/engine-smoke.mjs   # 真引擎烟测
node scripts/kit-check.mjs          # dsh-go plugin-kit（可选，需要 DSH_GO_KIT）
```

`DSH_ENGINE_DIR` 指向引擎安装根（里面要有 `node_modules/@deepseek-ai/cordis`）。桌面版安装目录通常是 `<安装目录>/resources/app/dsh`，手机本机引擎通常是 `/opt/dshapp/runtime`。

## 改动原则

1. **只用官方扩展点**：`systemPrompt.section()`、`system-prompt/assemble` waterfall、`agent/pre-step`、`agent/status`、`tools`、`commands`、`dshGo` 桥接。不要改引擎文件、不要解 asar、不要引入原生模块。
2. **零依赖**：`package.json` 不允许出现 `dependencies`。需要的能力优先用 Node 内置模块实现。
3. **Android 优先**：PRoot 上写盘很贵、内存有限。新增落盘要节流；新增常驻数据结构要能说明体积。
4. **没有桥接也要能跑**：`dshGo` 一律走 `ctx.inject(['dshGo'])` 局部等待，没有桥接的宿主（桌面版、SSH 主机）只少了界面，功能不能缺。
5. **撞名只警告**：与桌面版 dsh-purge 同装时不能拖垮加载。

## 提交

- 一个提交做一件事；提交信息用 `模块: 做了什么` 的形式（例：`recover: 账本按 turn/end 的 seq 去重`）。
- 改了行为就同步改 `README.md` / `docs/COMPATIBILITY.md` / `CHANGELOG.md` 的「未发布」段。
- 改了配置键，`README.md` 的配置表与 `lib/index.js` 的 `DEFAULTS` 必须一致。
- 新增文件记得进 `package.json` 的 `files` 白名单，否则 `npm pack` 不会带上它——这是最常见的漏。

## 提 PR 前自查

```bash
node --test test/*.test.js
DSH_ENGINE_DIR=<引擎根> node scripts/engine-smoke.mjs
bash scripts/build-selfinstall.sh        # 会顺带做一次真装自检
```

三条都过再提交。涉及界面的改动请附一张手机端截图（状态条 / 面板 / 工具卡）。

## 许可

本项目以 **AGPL-3.0** 发布。提交即表示你同意以同一许可授权你的贡献，并确认你有权提交这部分代码。引用第三方代码/正文时，请在 PR 里说明来源与许可。

## 安全问题

不要开公开 issue，按 [SECURITY.md](SECURITY.md) 私下报告。
