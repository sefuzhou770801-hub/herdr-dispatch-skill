---
name: herdr-dispatch
description: 用 herdr 把编码任务派给 codex 窗格并行施工——写任务书、派单、验收、回收窗格。首次使用自动完成 herdr 安装与配置。当用户提到 herdr、派单、开 codex 窗格干活、多 agent 并行施工时使用。
---

## 角色分工

全局只有两个角色：

- 规划 agent（你，herdr 里的主对话窗格）：盘清需求、写任务书、派单、验收、提炼经验。
- 执行 agent（codex 窗格）：无记忆、无全局上下文，任务书就是它的全部世界。

一单一个新窗格。派新单起新窗格，不往旧闲置窗格里塞——旧上下文会污染新任务。

## 首次配置

`herdr --version` 正常输出且 `~/.claude/bin/herdr-dispatch` 存在则跳过本节，直接进入派单流程。

1. 安装 herdr：`curl -fsSL https://herdr.dev/install.sh | sh`。源码仓库在 https://github.com/ogulcancelik/herdr （需要考察实现细节时克隆这个仓库），正式文档在 https://herdr.dev/docs/ 。
2. 依赖：`jq`（没有就 `brew install jq`）、codex CLI（没有就 `npm i -g @openai/codex`，并请用户完成 `codex login`）。
3. 安装集成：`herdr integration install claude` 和 `herdr integration install codex`，然后 `herdr integration status` 确认。集成让 herdr 拿到权威的 agent 状态上报，不装就只能靠读屏猜测，误判会多。
4. 安装 herdr 官方操作技能（教 agent 用 herdr CLI 的基础操作，本 skill 只管派单流程）：`npx skills add ogulcancelik/herdr --skill herdr -g`。
5. 把本 skill `scripts/` 目录下的三个脚本复制到 `~/.claude/bin/` 并 `chmod +x`（herdr-dispatch 派单、herdr-layout 归位窗格布局、herdr-idle 盘点闲置窗格）。
6. 请用户自己开一个终端窗口运行 `herdr` 进入界面（herdr 禁止嵌套启动，你不能替他运行），在其中一个窗格里启动 `claude` 作为日常主对话格。
7. 可选：在 `~/.claude/settings.json` 加 SessionStart hook 执行 `~/.claude/bin/herdr-idle --brief`，每次开会话自动报闲置窗格欠账。

完成标准：`herdr integration status` 显示 claude 与 codex 均已安装；`~/.claude/bin/herdr-dispatch` 可执行；用户已在 herdr 窗格内启动 claude（该会话里 `echo $HERDR_ENV` 输出 1）。

## 派单流程

1. 写任务书，存成文件（模板见 [references/task-template.md](references/task-template.md)）。四要素缺一不可，因为执行 agent 读不到你的任何上下文：背景（项目是什么、相关决策与约束）、任务（做什么、边界在哪）、输入（文件路径、命令、数据位置）、验收（产出物路径、达标标准）。
2. 派单：

   ```
   ~/.claude/bin/herdr-dispatch -n <任务名> -f <任务书路径> -c <工作目录>
   ```

   多个 agent 要动同一仓库时改用隔离模式 `-b <新分支> -r <仓库根>`（内部走 git worktree，从 origin/main 切新分支，工作树放 `~/projects/worktrees/`）——严禁两个 agent 共用一个工作目录。任务名必须唯一，不加「任务·」这类类别前缀（侧栏第一行直接显示名字，前缀只占宽度）。脚本已内置：任务书自动附加防转包条款和交付要求、原子发送防卡输入框、启动竞态重发、进入 working 的确认。退出码 2 = 未确认进入 working，`herdr agent read <任务名>` 人工查看画面。
3. 后台挂 `herdr agent wait <任务名> --status idle`（用 run_in_background，不阻塞对话、不轮询），收工自动唤醒。
4. 验收：idle 只代表停了，不代表做对了。`herdr agent read <任务名>` 核对产出物路径与执行证据；不达标就用 `herdr pane run <pane_id> "<返工要求>"` 打回重做。
5. 回收：验收完毕立即 `herdr pane close <pane_id>`（pane_id 用 `herdr agent get <任务名>` 现查），闲置窗格占内存、占额度会话。执行产出里的真经验（环境坑、命令用法）由你提炼记录，执行 agent 不写任何记忆。

## 规则与教训

- 寻址一律用 agent 名。窗格关闭后 workspace/tab/pane 的 id 会重排，不可当持久 id，名字才稳定；pane_id 只在使用前用 `herdr agent get <名>` 现查。
- 定位自己读 `HERDR_WORKSPACE_ID` / `HERDR_TAB_ID` / `HERDR_PANE_ID` 环境变量，严禁用 `focused: true` 定位自己——focused 是用户肉眼正在看的窗格。
- 给 agent 窗格发文本只用 `herdr pane run`（文本+回车一次原子提交）；`herdr agent send` 对 codex 常把文本卡在输入框里不执行。
- 严禁调用 socket API 的 `layout.apply`——它的语义是按模板新建 tab 替换旧 tab，窗格连进程一起被关。排布窗格用 `herdr-layout`（主对话格居左约 40% 宽通高，agent 窗格右侧排均分网格；同 tab 内 `pane move` 无效，脚本已内置先移出中转 tab 再搬回）。
- 关工作区里最后一个窗格 = 关掉整个工作区，单窗格常驻工作区严禁误伤。`herdr-idle` 盘点闲置窗格（`--close` 批量回收），默认跳过单窗格工作区。
- 状态识别存疑时用 `herdr agent explain <名> --json` 看判定证据；检测清单确实落后于 agent 界面改版时，写 `~/.config/herdr/agent-detection/<agent>.toml` 本地覆盖，`herdr server reload-agent-manifests` 生效，上游修复后删掉覆盖。
- 检查任务书是否送达不能用 `wait output`——它只等新出现的文本，秒送达反而误判为未送达。正确判定（脚本已内置）：状态 blocked = 停在确认页要人工；画面里有任务书路径 = 送达但没动，要人工；两者都不是 = 文本被启动期吞掉，重发。
- codex 启动报「未知参数」时，把 `herdr-dispatch` 脚本里 argv 的对应参数删掉再试——不同版本的 flag 有出入。
