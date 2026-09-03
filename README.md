# zach-skills

我自己在用的 [Claude Code](https://claude.com/claude-code) skills。

每个 skill 就是一个目录 + 一份 `SKILL.md`（YAML frontmatter 里的 `name` / `description` 决定它什么时候被触发）。仓库按**领域分层**，一层一个目录，目录内每个 skill 独立自足。

## 目录

| 层 | 说明 | Skills |
| --- | --- | --- |
| [`herdr/`](herdr/) | 在 [Herdr](https://herdr.dev) 终端复用器里编排并行 agent | `herdr-dispatch`、`herdr-monitor` |
| [`normal/`](normal/) | 不绑定外部工具的日常 skill，目前是两种解释语域 | `explaining-mechanisms-progressively`、`human-words` |

## 安装

Claude Code 从 `~/.claude/skills/<name>/SKILL.md` 加载用户级 skill，目录名即 skill 名。软链过去即可，改动实时生效：

```bash
git clone https://github.com/wine-fall/zach-skills.git ~/src/zach-skills

# 单个
ln -s ~/src/zach-skills/herdr/herdr-dispatch ~/.claude/skills/herdr-dispatch

# 某一层全装
for d in ~/src/zach-skills/herdr/*/; do
  ln -sfn "$d" ~/.claude/skills/"$(basename "$d")"
done
```

装好后 `/herdr-dispatch`、`/herdr-monitor` 这样直接调用，或者让 Claude 按 `description` 自动触发。

## 约定

- 一个 skill 一个目录，入口固定为 `SKILL.md`；辅助脚本、参考文档放同目录下。
- `description` 写清楚**什么时候该触发、什么时候不该触发** —— 这是模型唯一的路由依据。
- 分层目录只做归类，不影响加载：skill 名来自安装时的目录名，跟它在本仓库里的层级无关。
