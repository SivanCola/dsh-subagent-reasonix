# Reasonix 子智能体（DeepSeek Harness 插件）

把单个任务从 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）委派给本机的 [Reasonix](https://github.com/esengine/DeepSeek-Reasonix)。插件页显示为「Reasonix 子智能体」，用法与官方的「Codex 子智能体」「Claude Code 子智能体」一致。

*A DeepSeek Harness plugin that delegates one task to your local Reasonix over the Agent Client Protocol, using Reasonix's native authentication and settings.*

## 前置条件

- DeepSeek Harness **0.2.1-rc.2**（见[兼容性](#兼容性)）
- 已安装 Reasonix，并且 DSH 宿主能在 PATH 中找到 `reasonix`（Desktop 会读取登录 shell 的 PATH）
- Reasonix 已配置好模型与密钥（`reasonix setup`）。可用 `reasonix doctor` 检查。

## 安装

**Desktop / Web 插件页**：插件 → 添加插件 → 安装第三方插件，填入：

```
github:SivanCola/dsh-subagent-reasonix
```

**命令行**：

```sh
dsh plugin --profile <profile> add github:SivanCola/dsh-subagent-reasonix
```

装完后重启该 profile（Desktop 需重启应用）。插件默认启用，会给所有 agent 增加全局工具 `subagent_reasonix`。

## 使用

在对话里让 DSH 把任务交给 Reasonix，最好点名工具：

```
用 subagent_reasonix 让 Reasonix 去做：给 src/utils/date.ts 的 formatDate 补单元测试，跑通 npm test 后告诉我结果。
```

1. DSH 调用 `subagent_reasonix`，先返回子智能体 id。
2. Reasonix 在当前会话的工作目录里新建会话，独立完成任务。
3. 完成后，Reasonix 的最终答案作为通知回到对话，DSH 继续回复。

Reasonix 看不到你和 DSH 的对话历史，任务要写完整：路径、要求、怎样算完成。每次委派都是全新会话，不能追问；中间步骤记录在 `~/.reasonix/sessions/`，不回传 DSH。

## 工作方式

纯配置 bundle，没有 JS 代码。`cordis.patch.yml` 插入两行：

- `subagent-reasonix`：DSH 的通用 ACP 提供方 `@deepseek-ai/dsh-subagent-acp`，每次委派启动一个新的 `reasonix acp` 进程。
- `tool-subagent-reasonix`：`@deepseek-ai/dsh-tool-subagent`，把它暴露为全局工具 `subagent_reasonix`。

Reasonix 读取自己的 `~/.reasonix` 配置与密钥，所以模型、provider、沙箱、写入范围都沿用 Reasonix 原生设置。ACP 提供方会清理父进程里疑似凭据的环境变量，不影响 Reasonix。

## 调整配置

在 profile 的 `cordis.patch.yml` 中按 id 覆盖。覆盖会替换整个 `config`，需写全：

```yaml
- id: subagent-reasonix
  config:
    providerName: reasonix
    command: /absolute/path/to/reasonix   # PATH 里找不到 reasonix 时
    args: [acp, --workspace-only, --sandbox-bash, enforce]   # 收紧写入范围与 bash 沙箱
    permission: allow
```

`args` 可用 `reasonix acp --help` 中的参数，例如 `--model`、`--profile economy|balanced|delivery`。

## 安全

`permission: allow` 会自动批准 Reasonix 的所有权限请求（子智能体无人值守），实际边界取决于 Reasonix 自身的沙箱与写入范围设置。如果你的 Reasonix 关闭了 bash 沙箱，子智能体执行的命令不受沙箱限制；需要时按上文加 `--workspace-only --sandbox-bash enforce`。

## 兼容性

本插件的可用范围由 `@deepseek-ai/dsh-subagent-acp` 决定，`package.json` 里有三处相关声明，各自作用不同：

| 字段 | 值 | 作用 |
| --- | --- | --- |
| `engines.dsh` | `0.2.1-rc.2` | DSH 启动和安装时据此拒绝不匹配的宿主。存在此字段时，下面的 `peerDependencies` 不再参与判定。 |
| `dependencies` | `@deepseek-ai/dsh-subagent-acp` = `0.2.1-rc.2` | acp 提供方**不由 DSH 自带**，必须由插件引入。 |
| `peerDependencies` | `@deepseek-ai/dsh-tool-subagent` = `0.2.1-rc.2` | 该包由 DSH 自带，声明为 peer 以免装出第二份；`optional` 是因为宿主已提供，缺了也不阻止安装。 |

**为什么 acp 不能写成 peer。** DSH 本体、`dsh-base`、`dsh-web-app` 都不包含 `@deepseek-ai/dsh-subagent-acp`，只有 `dsh-tool-subagent` 是宿主自带的。把 acp 改成 peer 会让安装后的 profile 里无人提供该模块，启动时报 `Cannot find package '@deepseek-ai/dsh-subagent-acp'`。

**为什么版本必须跟着宿主走。** acp 把自己的 peer 精确钉死到同一发行版（即 `@deepseek-ai/dsh-session` = `0.2.1-rc.2`，不是范围）。宿主升级到 `0.2.1-rc.3` 后，`acp@0.2.1-rc.2` 就会被 DSH 拒绝挂载。所以：

- 用**精确版本**而不是范围：写成 `>=0.2.1-rc.2 <0.2.2-0` 时，pnpm 会解析到区间内最高的 `0.2.1-rc.5`，那个版本反而不被当前宿主接受，安装会被拒。
- DSH 升级后需要装对应新版本的插件，这是 acp 的发行节奏决定的，不是本插件额外加的限制。
- 升级后若版本不匹配，DSH 会在启动时明确报出该插件不兼容并跳过，而不是让 `subagent_reasonix` 悄无声息地消失。

## 已知问题

- [dsh-external/issues#668](https://github.com/dsh-external/issues/issues/668)：父代理不知道如何等待外部子代理，可能用 `sleep` 阻塞，导致结果晚到。可在请求里加一句「启动后直接结束本轮，等完成通知」。
- [dsh-external/issues#669](https://github.com/dsh-external/issues/issues/669)：DSH 的 ACP 提供方把子智能体的过程说明和最终答案拼在一起回传。

## 许可

[MIT](LICENSE)。`icon.svg` 改编自 Reasonix Studio 图标（MIT，© Reasonix Contributors），见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
