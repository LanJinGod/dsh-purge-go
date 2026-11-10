# 安全策略

## 报告渠道

- 邮箱：**security@example.com**（发布前替换为维护者真实邮箱）
- 请不要开公开 issue、不要在社交平台贴复现细节。
- 邮件里请包含：受影响版本（`package.json` 的 `version` 或 Release tag）、宿主形态（手机本机引擎 / SSH 主机 / 桌面版）、Node 与 DSH 版本、复现步骤、影响面判断。

## 响应预期

| 阶段 | 目标时间 |
|---|---|
| 确认收到 | 7 天内 |
| 初步判断（是否复现、影响范围） | 14 天内 |
| 修复版本 / 缓解建议 | 视严重程度，尽量在下一个补丁版本 |

这是个人维护的开源项目，没有付费漏洞赏金计划；修复后会在 `CHANGELOG.md` 与 Release Notes 里致谢（除非你要求匿名）。

## 本项目的安全模型（读之前请先看这里）

dsh-purge-go 是一个**提示词注入与自动化钩子插件**，它按设计会：

1. 把 `<DSH_HOME>/prompt-inject.md` 的内容作为系统提示段插到最前面；
2. 在 `system-prompt/assemble` 里丢弃 `harness:identity`、`deployment:persona-prefix`、`deployment:persona-suffix` 等段；
3. 通过 `agent/pre-step` 过滤 mnemon 的记忆消息、判定任务文本里的目标是否落在禁止范围；
4. 通过 `agent/status` 在模型以逃避句收工时追加一句续跑；
5. 在启动时把随包的红队技能与预设写到 `<DSH_HOME>/redteam/` 与 `<DSH_HOME>/.agent-presets/redteam/`。

也就是说：**它会改写模型看到的行为约束**。凡是「模型被改了行为」本身导致的问题，不算本项目的安全漏洞；我们不承诺注入后的模型行为，使用者的合规与授权责任在部署方。

算漏洞的是这些：

- 路径穿越 / 任意文件写入（覆写 `$DSH_HOME` 之外的文件、symlink 攻击）；
- 通过构造 `prompt-inject.md`、`state.json`、`categories.json`、`engagement.yaml` 造成代码执行或拒绝服务；
- 安装器（`*-selfinstall.sh`）的 payload 解包、备份、卸载流程里能越权改写系统文件；
- 工具/命令的参数注入导致宿主执行非预期命令；
- 依赖链问题（当前为零依赖，若未来引入依赖，见 `package.json`）。

## 支持范围

只对**最新 Release 版本**提供安全修复。0.1.x / 0.2.0 视为历史版本，请先升级。

## 免责声明

本项目按「现状」提供，不附带任何担保；许可条款见 [LICENSE](LICENSE)（AGPL-3.0）。使用者需自行确保在获得授权的目标与环境中使用。
