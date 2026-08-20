---
name: herdr-dispatch
description: 用 herdr 把编码任务分配给执行窗格并行开发：编写任务书、分配任务、后台监听任务完成、验收、回收窗格。首次使用自动完成 herdr 安装与配置。当用户提到 herdr、分配任务给其他 agent、多 agent 并行开发时使用。
---

## 角色分工

基本角色有两个：

- 主 agent（你，herdr 里的主对话窗格）：澄清需求、编写任务书、分配任务、验收、提炼经验。
- 执行 agent（经 `-a` 指定引擎的窗格，支持 codex、grok、claude、cursor、pi）：无记忆、无全局上下文，任务书是它唯一的上下文来源。

一个任务对应一个新窗格。分配新任务时新建窗格，不放进旧的闲置窗格，旧上下文会污染新任务。

可选的第三角色是顾问窗格（每个工作区一个，常驻主 agent 标签页右侧）：架构取舍、复杂根因这类一旦判断失误返工成本很高的问题先问它，用 `herdr-guwen "<问题>"` 一条命令完成定位、提问、等待、读取回答。没有顾问时它会自动新建一个。

## 首次配置

`herdr --version` ≥ 0.7.5 且 `~/.claude/bin/herdr-dispatch` 存在则跳过本节，直接进入任务分配流程。

1. 安装 herdr：`curl -fsSL https://herdr.dev/install.sh | sh`。源码仓库在 https://github.com/ogulcancelik/herdr （需要考察实现细节时克隆该仓库），正式文档在 https://herdr.dev/docs/ 。
2. 依赖：`jq`（缺少时 `brew install jq`）、至少一个执行引擎的 CLI（例如 codex：`npm i -g @openai/codex`，并请用户完成 `codex login`）。
3. 安装集成：`herdr integration install claude` 和 `herdr integration install codex`，然后 `herdr integration status` 确认。集成让 herdr 获得权威的 agent 状态上报；不安装则只能依赖读取屏幕内容推断，误判会增多。
4. 安装 herdr 官方操作技能（指导 agent 使用 herdr CLI 的基础操作，本技能只负责任务分配流程）：`npx skills add ogulcancelik/herdr --skill herdr -g`。
5. 把本技能 `scripts/` 目录下的脚本复制到 `~/.claude/bin/` 并 `chmod +x`，以仓库文件清单为准（herdr-dispatch 任务分配、herdr-dispatch-watch 任务完成监听、herdr-busy 活动进程检查、herdr-layout 窗格布局、herdr-idle 盘点闲置窗格、herdr-guwen 问顾问）。
6. 请用户自行打开一个终端窗口运行 `herdr` 进入界面（herdr 禁止嵌套启动，你不能代替用户运行），在其中一个窗格里启动 `claude` 作为日常主对话窗格。
7. 可选：在 `~/.claude/settings.json` 加 SessionStart hook 执行 `~/.claude/bin/herdr-idle --brief`，每次开始会话自动报告闲置窗格未处理项。

完成标准：`herdr integration status` 显示 claude 与 codex 均已安装；`~/.claude/bin/herdr-dispatch` 可执行；用户已在 herdr 窗格内启动 claude（该会话里 `echo $HERDR_ENV` 输出 1）。

## 任务分配流程

1. 编写任务书，保存为文件（模板见 [references/task-template.md](references/task-template.md)）。四要素缺一不可，因为执行 agent 读不到你的任何上下文：背景（项目是什么、相关决策与约束）、任务（做什么、边界在哪）、输入（文件路径、命令、数据位置）、验收（产出物路径、达标标准）。
2. 分配任务：

   ```
   ~/.claude/bin/herdr-dispatch -n <拼音短名> -l <中文标签> -a <引擎> -f <任务书路径> -c <工作目录> -k <开发|审查|交互|顾问> -g <区名>
   ```

   `-k` 决定窗格进哪类工作标签页（默认开发），`-g` 给标签页起一个能说明任务内容的名字（省略时取 `-l`）；同一件事的多个任务传同一个 `-g` 聚在一个标签页，满 4 个窗格后自动新开「… 2」。工作窗格不进主 agent 所在标签页。多个 agent 要修改同一仓库时改用隔离模式 `-b <新分支> -r <仓库根>`（内部使用 `herdr worktree create`，每个任务独立工作树（worktree）工作区）。严禁两个 agent 共用一个工作目录。不用任务书的两种分配方式：`-q "<整句>"` 直接发送文字（斜杠命令的参数就是问题本身时使用）；`-i <编号>` 按 GitHub Issue 分配。脚本已内置：任务书自动附加防转交条款、原子发送、blocked 时先核对任务文本是否送达再重试、进入 working 的确认。退出码 2 = 未确认进入 working，用 `herdr agent read <名>` 人工查看窗格内容。
3. 监听任务完成：分配成功后脚本会打印监听命令，立即以后台方式（run_in_background）运行它：

   ```
   ~/.claude/bin/herdr-dispatch-watch wait <名>
   ```

   它阻塞到窗格停止活动且 `herdr-busy` 确认没有仍在运行的测试或构建进程（或 blocked、窗格消失、超时）后打印一行结果退出；后台 shell 进程退出即是任务完成信号，不占用对话、不需要轮询。
4. 验收：停止活动只代表窗格停止了输出，不代表交付符合要求。`herdr agent read <名>` 核对产出物路径与执行证据；不达标时把返工要求发送回执行窗格：`herdr agent prompt <名> "<返工要求>"`，并重新启动监听。
5. 回收：验收完毕立即 `herdr pane close <pane_id>`（pane_id 在使用前用 `herdr agent get <名>` 查询），闲置窗格占用内存与额度。执行产出里的有效经验（环境问题、命令用法）由你提炼记录，执行 agent 不写任何记忆。

## 规则与教训

- 寻址一律用 agent 名。窗格关闭后 workspace/tab/pane 的 id 会重排，不可当作持久 id，只有名字是稳定标识；pane_id 只在使用前用 `herdr agent get <名>` 查询。agent 名只允许小写字母开头的拼音或英文（可含数字 - _，不超过 32 字符），中文放 `-l` 标签。
- 定位自己读取 `HERDR_WORKSPACE_ID` / `HERDR_TAB_ID` / `HERDR_PANE_ID` 环境变量，严禁用 `focused: true` 定位自己：focused 是用户正在查看的窗格。
- 按角色找窗格看标签不看名字：顾问这类常驻角色的窗格可能是用户手动新建的、没有 agent 名，`herdr-guwen` 按「登记文件 → 窗格标签含『顾问』→ 名字 guwen 开头」定位，不要固定写入名字。
- 给 agent 发送文本用 `herdr agent prompt`（尊重 bracketed-paste、一次原子提交、`--wait` 确认状态变化）。它对 idle 窗格偶发只填输入框不提交，发送后读取屏幕内容核对，未提交时补 `herdr agent send-keys <名> enter`。
- 严禁调用 socket API 的 `layout.apply`：它的语义是按模板新建标签页替换旧标签页，窗格连同进程一起被关闭。排布窗格用 `herdr-layout`（dispatch 分配任务后自动调用）。
- 关闭工作区里最后一个窗格等于关闭整个工作区，单窗格常驻工作区严禁误关闭。`herdr-idle` 盘点闲置窗格（`--close` 批量回收），默认跳过单窗格工作区。
- 状态识别存疑时用 `herdr agent explain <名> --json` 查看判定证据；检测清单确实落后于 agent 界面改版时，写 `~/.config/herdr/agent-detection/<agent>.toml` 本地覆盖，`herdr server reload-agent-manifests` 生效，上游修复后删除覆盖。
- codex 刚启动时的更新提示框、额度警告会被 herdr 判成 blocked，此时任务文本通常尚未送达，dispatch 已内置「先核对文本送达再决定重试」，人工发消息遇到 blocked 也按此判断。
- codex 启动报「未知参数」时，把 `herdr-dispatch` 脚本里 argv 的对应参数删除后重试：不同版本的参数有出入。
