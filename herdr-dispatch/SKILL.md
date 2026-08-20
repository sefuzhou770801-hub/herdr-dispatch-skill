---
name: herdr-dispatch
description: 用 herdr 把编码任务派给 codex 窗格并行开发：写任务书、派单、后台监听完工、验收、回收窗格。首次使用自动完成 herdr 安装与配置。当用户提到 herdr、派单、开 codex 窗格干活、多 agent 并行开发时使用。
---

## 角色分工

全局只有两个角色：

- 主 agent（你，herdr 里的主对话窗格）：盘清需求、写任务书、派单、验收、提炼经验。
- 执行 agent（codex 窗格）：无记忆、无全局上下文，任务书就是它的全部世界。

一单一个新窗格。派新单起新窗格，不往旧闲置窗格里塞，旧上下文会污染新任务。

可选的第三角色是顾问窗格（每个工作区一个，常驻主 agent 标签页右侧）：架构取舍、复杂根因这类「做错了返工成本高」的判断先问它，用 `herdr-guwen "<问题>"` 一条命令完成定位、提问、等待、读回回答。没有顾问时它会自动起一个。

## 首次配置

`herdr --version` ≥ 0.7.5 且 `~/.claude/bin/herdr-dispatch` 存在则跳过本节，直接进入派单流程。

1. 安装 herdr：`curl -fsSL https://herdr.dev/install.sh | sh`。源码仓库在 https://github.com/ogulcancelik/herdr （需要考察实现细节时克隆这个仓库），正式文档在 https://herdr.dev/docs/ 。
2. 依赖：`jq`（没有就 `brew install jq`）、codex CLI（没有就 `npm i -g @openai/codex`，并请用户完成 `codex login`）。
3. 安装集成：`herdr integration install claude` 和 `herdr integration install codex`，然后 `herdr integration status` 确认。集成让 herdr 拿到权威的 agent 状态上报，不装就只能靠读屏猜测，误判会多。
4. 安装 herdr 官方操作技能（教 agent 用 herdr CLI 的基础操作，本 skill 只管派单流程）：`npx skills add ogulcancelik/herdr --skill herdr -g`。
5. 把本 skill `scripts/` 目录下的六个脚本复制到 `~/.claude/bin/` 并 `chmod +x`（herdr-dispatch 派单、herdr-dispatch-watch 完工监听、herdr-busy 活动进程检查、herdr-layout 归位窗格布局、herdr-idle 盘点闲置窗格、herdr-guwen 问顾问）。
6. 请用户自己开一个终端窗口运行 `herdr` 进入界面（herdr 禁止嵌套启动，你不能替他运行），在其中一个窗格里启动 `claude` 作为日常主对话格。
7. 可选：在 `~/.claude/settings.json` 加 SessionStart hook 执行 `~/.claude/bin/herdr-idle --brief`，每次开会话自动报闲置窗格欠账。

完成标准：`herdr integration status` 显示 claude 与 codex 均已安装；`~/.claude/bin/herdr-dispatch` 可执行；用户已在 herdr 窗格内启动 claude（该会话里 `echo $HERDR_ENV` 输出 1）。

## 派单流程

1. 写任务书，存成文件（模板见 [references/task-template.md](references/task-template.md)）。四要素缺一不可，因为执行 agent 读不到你的任何上下文：背景（项目是什么、相关决策与约束）、任务（做什么、边界在哪）、输入（文件路径、命令、数据位置）、验收（产出物路径、达标标准）。
2. 派单：

   ```
   ~/.claude/bin/herdr-dispatch -n <拼音短名> -l <中文标签> -a codex -f <任务书路径> -c <工作目录> -k <开发|审查|交互> -g <区名>
   ```

   `-k` 决定窗格进哪类工作标签页（默认开发），`-g` 给标签页起个说得清在干什么的名字（省略取 `-l`）；同一件事的多个任务传同一个 `-g` 聚在一个标签页，满 4 格自动开「… 2」。工作窗格不进主 agent 所在标签页。多个 agent 要动同一仓库时改用隔离模式 `-b <新分支> -r <仓库根>`（内部走 `herdr worktree create`，每个任务独立 worktree 工作区）。严禁两个 agent 共用一个工作目录。不用任务书的两种派法：`-q "<整句>"` 直接发送文字（斜杠命令的参数就是问题本身时用）；`-i <编号>` 按 GitHub Issue 派单。脚本已内置：任务书自动附加防转交条款、原子发送、blocked 时先核对任务文本是否送达再重试、进入 working 的确认。退出码 2 = 未确认进入 working，`herdr agent read <名>` 人工查看画面。
3. 监听完工：派单成功后脚本会打印监听命令，立即以后台方式（run_in_background）运行它：

   ```
   ~/.claude/bin/herdr-dispatch-watch wait <名>
   ```

   它阻塞到窗格停止活动且 `herdr-busy` 确认没有仍在运行的测试或构建进程（或 blocked、窗格消失、超时）后打印一行结果退出；后台 shell 退出即是完工信号，不占对话、不用轮询。
4. 验收：停止活动只代表停了，不代表做对了。`herdr agent read <名>` 核对产出物路径与执行证据；不达标就把返工要求发回去：`herdr agent prompt <名> "<返工要求>"`，再挂一次监听。
5. 回收：验收完毕立即 `herdr pane close <pane_id>`（pane_id 用 `herdr agent get <名>` 现查），闲置窗格占内存、占额度会话。执行产出里的真经验（环境坑、命令用法）由你提炼记录，执行 agent 不写任何记忆。

## 规则与教训

- 寻址一律用 agent 名。窗格关闭后 workspace/tab/pane 的 id 会重排，不可当持久 id，名字才稳定；pane_id 只在使用前用 `herdr agent get <名>` 现查。agent 名只允许小写字母开头的拼音或英文（可含数字 - _，不超过 32 字符），中文放 `-l` 标签。
- 定位自己读 `HERDR_WORKSPACE_ID` / `HERDR_TAB_ID` / `HERDR_PANE_ID` 环境变量，严禁用 `focused: true` 定位自己：focused 是用户肉眼正在看的窗格。
- 按角色找窗格看标签不看名字：顾问这类常驻角色的窗格可能是用户手动起的、没有 agent 名，`herdr-guwen` 按「登记文件 → 窗格标签含『顾问』→ 名字 guwen 开头」定位，不要写死名字。
- 给 agent 发文本用 `herdr agent prompt`（尊重 bracketed-paste、一次原子提交、`--wait` 确认状态变化）。它对 idle 窗格偶发只填输入框不提交，发送后读屏核对，没提交就补 `herdr agent send-keys <名> enter`。
- 严禁调用 socket API 的 `layout.apply`：它的语义是按模板新建 tab 替换旧 tab，窗格连进程一起被关。排布窗格用 `herdr-layout`（dispatch 派单后自动调用）。
- 关工作区里最后一个窗格 = 关掉整个工作区，单窗格常驻工作区严禁误伤。`herdr-idle` 盘点闲置窗格（`--close` 批量回收），默认跳过单窗格工作区。
- 状态识别存疑时用 `herdr agent explain <名> --json` 看判定证据；检测清单确实落后于 agent 界面改版时，写 `~/.config/herdr/agent-detection/<agent>.toml` 本地覆盖，`herdr server reload-agent-manifests` 生效，上游修复后删掉覆盖。
- codex 刚启动时的更新提示框、额度警告会被 herdr 判成 blocked，此时任务文本多半没送进去，dispatch 已内置「先核对文本送达再决定重试」，人工发消息遇到 blocked 也照此判断。
- codex 启动报「未知参数」时，把 `herdr-dispatch` 脚本里 argv 的对应参数删掉再试：不同版本的 flag 有出入。
