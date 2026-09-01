# herdr

在 [Herdr](https://herdr.dev)（面向 coding agent 的终端复用器）里把任务分派给并行的 Claude Code 会话，并把它们盯到收工。

两个 skill 都要求运行在 Herdr 内部（`HERDR_ENV=1`），否则直接停手，不从外部去遥控 Herdr。

| Skill | 干什么 |
| --- | --- |
| [`herdr-dispatch`](herdr-dispatch/) | 把当前会话已经谈定的方向写成自足的 brief，开一个同级 pane，起一个新的 Claude Code，把 brief 原子地投进去，然后交出 handle 就撒手。 |
| [`herdr-monitor`](herdr-monitor/) | 后台盯着本会话派出去的 agent，谁停下来就读、判断、再推一把。只有两种情况才回头找人：子 agent 缺一个它自己给不了的权限，或者遇到真正重大 / 不可逆的决定。 |

## 两者的分工

`dispatch` 的契约到「报出 handle」为止 —— **它绝不自动接 `monitor`**。想盯着，是一次单独的、明确的请求（`/herdr-monitor`）。这条边界写在 `herdr-monitor` 的第 0 节，是刻意的：派活和看活是两件事，混在一起会让每次派活都拖出一个没人要的后台监控。

## 共同的坑

- **分组前缀**：一个主会话派出去的所有 agent 共用一个前缀（取自派发方 tab 的 label），所以 `herdr agent list | grep '<prefix>/'` 能一把捞出来。前缀不从任务名推导 —— 任务会变，会话不会。
- **投递要原子**：用 `herdr agent prompt --wait --until working`，别用 send-keys 拼。`--wait` 超时 ≠ 没送到，先 `agent get` 再说，盲目重发会投两遍。
- **composer 里的文字不是用户输入**：Claude Code 会在输入行渲染 ghost 提示文本，`read` 分辨不出来。别把它当成人类打的字，也别对不是自己注入的文本按回车。
