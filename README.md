# herdr-dispatch skill

把编码任务派给 [herdr](https://github.com/ogulcancelik/herdr) 里的 codex 窗格并行开发的 Claude Code 技能：写任务书、派单、后台监听完工、验收、回收窗格，首次使用自动完成 herdr 安装与配置。

## 安装

```bash
git clone https://github.com/sefuzhou770801-hub/herdr-dispatch-skill.git
cp -R herdr-dispatch-skill/herdr-dispatch ~/.claude/skills/
```

然后对 Claude Code 说一句「按 herdr-dispatch 技能把环境配齐」，剩下的交给它。

## 内容

- `herdr-dispatch/SKILL.md`：角色分工、首次配置、派单流程、规则与教训
- `herdr-dispatch/scripts/herdr-dispatch`：派单脚本（任务书附加防转交条款、原子发送、blocked 先核对送达再重试、进入 working 的确认、按 `-k` 分区落工作标签页、`-b/-r` worktree 隔离模式）
- `herdr-dispatch/scripts/herdr-dispatch-watch`：完工监听。`wait <名>` 阻塞到窗格停止活动且无受监控进程后退出，主 agent 以后台 shell 运行它，shell 退出即完工信号
- `herdr-dispatch/scripts/herdr-busy`：判断窗格里是否仍有测试或构建进程在运行，避免只凭终端无输出判断完工
- `herdr-dispatch/scripts/herdr-layout`：窗格归位（主对话格左 40%，工作标签页均分网格）
- `herdr-dispatch/scripts/herdr-idle`：闲置窗格盘点与批量回收
- `herdr-dispatch/scripts/herdr-guwen`：向本工作区的顾问窗格提问并读回回答，按角色定位不依赖名字
- `herdr-dispatch/references/task-template.md`：任务书四要素模板
