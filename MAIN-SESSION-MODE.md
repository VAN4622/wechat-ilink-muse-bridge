# 主会话工具模式（本仓库的变体）

桥的代码和上游 [gfydfhcf/wechat-ilink-muse-bridge](https://github.com/gfydfhcf/wechat-ilink-muse-bridge) 一致，许可证见 LICENSE。这个变体只改一件事：微信消息由谁处理。

## 上游模式的问题

按 AGENTS.md 的默认契约，hook 每来一批消息就唤醒一个 worker 去处理。问题是这个 worker 不是主会话，它看不到主会话里聊过的内容，主会话也不知道微信里发生过什么。对"一个窗口长周期使用"的个人 agent 来说，两边上下文对不上，体验是割裂的。

## 工具模式怎么做

微信只当主 agent 的一个外部工具：消息流转是通的，UI 不追求同步。

- hook 的 worker 降为纯信使：原子认领批次，把每条消息逐字转录进执行总结，递给主会话。它不写回复、不调用 send、不打开媒体解读、不守候等待。指令模板在 `examples/muse-hook/courier-prompt.md`。
- 主 agent 本人按 `skill/wechat_tool/` 里的规则处理这批消息：读内容、想回复、调 `ilink.py send` 回到微信。回复的是主会话这个 agent，记得主会话的全部上下文。

## 四条硬约束

- 来源明确：hook 递进来的批次就是微信来源，哪怕它落在主窗口里。微信来的消息只用微信 send 回，不拿主窗口的气泡当回复；用户在主窗口说话，才在主窗口回。
- 去重：批次首条 message_id 是整批的键。回复前查发送账本（`data/send_ledger.json`），键在就整批跳过；send 带 `--key <首条 message_id>`，多气泡用 `-1`、`-2` 后缀；发送失败不手动重试，客户端自己重试一次。chatlog 每批只写一条。
- 默认静默：处理完之后主窗口不留处理记录，直接结束回合。只有发送失败、桥的故障告警、或用户在主窗口问起时，主窗口才出声。
- 串行：主 agent 忙着跑别的回合时，微信消息排队等。个人 agent 场景下这是可接受的，甚至更符合直觉；代价是回复延迟在几秒到半分钟，不是秒回。

## 部署差异

其余步骤按 AGENTS.md 走，只有这几处不同：

1. AGENTS.md 第 1、1.5、3 步照旧：扫码登录、绑定 owner、验证收发。
2. 第 2 步选 hook 驱动，脚本仍用 `examples/muse-hook/ilink-watch.sh`，但 hook 定义里的 worker 指令换成 `examples/muse-hook/courier-prompt.md` 的信使版（替换里面的路径占位符）。不要用上游"醒来就处理批次"的默认契约，也不要运行 `ilink-warm-wait.sh`。
3. 把 `skill/wechat_tool/` 放进你的 Muse 自定义 skill 目录，并让它在每轮会话加载（skill.yaml 里 `include_in_prompt: true`）。
4. watchdog 和 chatlog 轮转的定时任务照 AGENTS.md 装。

## 前提

- 你的 agent 有一个长期存在的主会话，hook 的执行总结能递到主会话（Muse 的 hook 驱动就是这样）。
- 登录凭证 `data/session.json` 永远只留在本机，不要提交、不要外传，仓库的 .gitignore 已经排除了整个 `data/`。
