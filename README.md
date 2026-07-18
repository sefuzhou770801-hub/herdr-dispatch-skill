# herdr-dispatch skill

把编码任务派给 [herdr](https://github.com/ogulcancelik/herdr) 里的 codex 窗格并行施工的 Claude Code 技能：写任务书、派单、验收、回收窗格，首次使用自动完成 herdr 安装与配置。

## 安装

```bash
git clone https://github.com/sefuzhou770801-hub/herdr-dispatch-skill.git
cp -R herdr-dispatch-skill/herdr-dispatch ~/.claude/skills/
```

然后对 Claude Code 说一句「按 herdr-dispatch 技能把环境配齐」，剩下的交给它。

## 内容

- `herdr-dispatch/SKILL.md` — 角色分工、首次配置、派单流程、规则与教训
- `herdr-dispatch/scripts/herdr-dispatch` — 派单脚本（任务书附加防转包条款、原子发送、启动竞态重发、进入 working 的确认）
- `herdr-dispatch/scripts/herdr-layout` — 窗格归位（主对话格左 40%，agent 右侧网格）
- `herdr-dispatch/scripts/herdr-idle` — 闲置窗格盘点与批量回收
- `herdr-dispatch/references/task-template.md` — 任务书四要素模板
