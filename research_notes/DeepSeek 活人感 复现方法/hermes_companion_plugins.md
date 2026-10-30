# Hermes Agent 专用的人格 / 陪伴 / “活人感”插件与配置调研（含开放缺口源码核查）

> 调研日期 2026-10-30。Hermes Agent 源码引用固定在 tag `v0.22.5`（发布于 2026-10-23T05:28Z），main 分支最后推送时间是 2026-10-30T06:49Z（[repo](https://github.com/NousResearch/hermes-agent)）。所有第三方仓库都标了 commit。全程只读：只通过 `gh api` 或 raw 读取了源码，没有运行、fork、star 或发布任何内容。
> 说明：读 v0.22.5 的 `hooks.md` 时，raw.githubusercontent.com 返回了限流（429），所以 `transform_llm_output` 一节读的是同目录里另一份 main 分支副本，相应地引用 main 分支链接和官方文档站。

---

## 1. gejifeng/Hermes_Soul_patch：功能、使用的 hook、提示词，以及是否实现分段、延迟、情绪、主动消息、反助手腔

### Takeaway
Hermes_Soul_patch v0.5 是一个**独立的常驻服务**（SQLite + 本地 HTTP API + 自己起的 Hermes `AIAgent` worker），外加一个只挂 `pre_llm_call` / `post_llm_call` 两个 hook 的桥接插件。它实现的“活人感”分四块：
- 基于“联系间隔”事实的情绪（想念、失落、撒娇）；
- 兴趣驱动的自主“生活”（逛小红书），并且记忆必须有来源；
- 主动发起的一组 1–4 条短消息。这组消息绕过 Hermes 网关，直接调 Telegram Bot API，并按块定时发出；
- 一段专门的“去助手腔”表达规范（EXPRESSION_STYLE）。

它**没有**对 Hermes 网关里正常聊天的回复做分段，也没有打字延迟或 typing 指示。

### Cited Findings
- **基本信息**：描述是“Hermes companion plugin: 兴趣驱动的连续生活、有来源的记忆、联系感知与按块外发、主人审批的小红书互动”，MIT 许可，★17，fork 5，创建于 2026-05-19，最后推送 2026-10-24。仓库只有一个 commit `9233659`（2026-10-24），提交信息为 “Rewrite as Hermes Soul v0.5: interest-led life runtime”。— [repo](https://github.com/gejifeng/Hermes_Soul_patch)
- **v0.5 是重写版**。README 原文：“本仓库的 v0.5 是一次完整重写，取代了早期的 `hermes-companion` 原型（情感状态机、世界模拟）”。架构是“一个独立运行的服务，加上一个 Hermes 生命周期插件。Hermes 核心代码不需要任何改动”。— [README.zh.md](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/README.zh.md)
- **插件清单**：插件名 `hermes-soul-bridge`，`provides_hooks: [pre_llm_call, post_llm_call]`。— [hermes_plugin/plugin.yaml](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_plugin/plugin.yaml)
- **插件行为**：
  - 文件 docstring 写的是 `"No inference in hooks; no system prompt changes; no dependence on Cognitive."`。
  - `before()` 只处理 `platform == "telegram"` 且 `sender_id` 在 `bridge_owner_ids` 里的轮次，把本轮 POST 给本地服务 `/api/bridge`，再返回 `{"context": result["context"], "target": "user_message"}`。也就是说，生活背景注入到**用户消息**里，不动系统提示。
  - `after()` 在 `post_llm_call` 时把 `assistant_response` 回传给服务记账。
  - 非 default profile 直接 return，原因写在注释里：`"The autonomous worker profile must never feed its own generated prompts back as user facts."`
  - — [hermes_plugin/__init__.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_plugin/__init__.py)
- **后台 worker 的调用方式**：`worker.py` 用单独的 `HERMES_HOME`（一个专用 profile）直接构造 `AIAgent(...)`，参数包括：
  - `enabled_toolsets=[]`、`skip_memory=True`、`skip_context_files=True`、`max_iterations=2`；
  - `reasoning_config={"enabled": False, "effort": "none"}`；
  - `request_overrides=overrides`，开启 `structured_json` 时再加 `overrides["response_format"] = {"type": "json_object"}`。
  - — [hermes_soul/worker.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/worker.py)
- **示例配置里关掉思考的写法**：
  ```yaml
  request_overrides:
    extra_body:
      chat_template_kwargs:
        enable_thinking: false
  ```
  同一份配置里还有 `interval_seconds: 720`、`max_calls_per_day: 500`、`max_output_tokens: 1800`。— [deploy/config.example.yaml](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/deploy/config.example.yaml)
- **主动消息的分段与节奏**（配置注释原文）：“按块发送：块与块之间至少 min_interval_seconds；块内（如「在吗？」后再追一句）只需 block_gap_seconds，每块最多 max_per_block 条，全天最多 max_per_day 条。”默认值：
  - `min_interval_seconds: 3600`、`block_gap_seconds: 150`、`max_per_block: 4`、`max_per_day: 32`；
  - 免打扰 `quiet_start: 22`、`quiet_end: 8`；
  - 睡眠 `00:00–08:00`（Asia/Shanghai）；
  - 默认 `mode: draft` 加 `authorized: false`，也就是只留草稿、不发送。
  - — [deploy/config.example.yaml](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/deploy/config.example.yaml)
- **主动消息怎么发出去**：`Expressions.deliver_one()` 直接 `POST https://api.telegram.org/bot<token>/sendMessage`，一次发一条，**不经过 Hermes 网关**，也不自动重试（结果不确定时标为 `uncertain`）。`next_ready()` 的注释写的是 “Blocks are separated by min_interval_seconds; inside a block, block_gap_seconds suffices.”
  `delivery_block()` 在以下情况都会阻止外发：
  - 主人刚说过话（15 分钟内）；
  - 免打扰时段或睡眠时段；
  - 已达每日额度；
  - 交流处于 pause 边界期间。
  - — [hermes_soul/expressions.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/expressions.py)
- **REACH_OUT 提示词**（原文）：
  > `<<OWNER>>已经很久没消息了，你决定主动找<<OWNER>>。依据 life.contact 的实际间隔、未回复数和上次交流，写一组像真人发微信那样的短消息。`
  > `可以先「在吗？」再追一句，或者直接说想<<OWNER>>、问<<OWNER>>在干嘛、撒娇埋怨<<OWNER>>不理你；间隔越长、发了没回越多，情绪越明显。`
  > `只返回 JSON：{"messages":["第一条","第二条"], "reason":"简短缘由"}。messages 为 1 到 4 条，每条一句或两句；决定先不发时 messages 为空列表。`
  — [hermes_soul/prompts.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)
- **去助手腔规范 EXPRESSION_STYLE**（原文节选）：
  > `用自然、简短的中文，直接表达当下的感受、偏好、想法和需要。意思说完整就收句，结尾落在当前话题上。`
  > `写完后检查每句后半句与整段结尾：省去顺手补上的反向否认、自我辩解、劝阻和命令。保留已经说清的正面意思，必要的补充改为具体感受或邀请。关心疲惫时可以说「那就早点休息，我陪你待一会儿」……`
  > `偏好说明具体吸引点，关心说出具体关心，撒娇说出希望对方做的事。补充说明围绕用户实际提出的问题。`
  > `只有当前交流确有误解需要澄清、选项需要比较或边界需要表达时，才使用否定、对照或制止语气；一次说清即可。`

  这段规范同时拼进了 `IDENTITY_CONTRACT`、`REACH_OUT` 和桥接上下文（`bridge_context`）。— [hermes_soul/prompts.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)；[hermes_soul/runtime.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/runtime.py)
- **情绪 / 联系感知提示词**（IDENTITY_CONTRACT v0.4 原文）：
  > `想<<OWNER>>是自然的；平静默认状态不适用于<<OWNER>>长时间不理你。随着间隔变长，情绪可以越来越明显：想念、失落、委屈、撒娇式埋怨、闹点小脾气；发了消息没回会更在意。`
  > `情绪说出口就好，不用威胁、冷暴力、自我伤害或长期惩罚性的话，也不替<<OWNER>>编造不回的理由。<<OWNER>>回来时可以先带着情绪，再看<<OWNER>>的态度慢慢软下来。`

  v0.3 还有一句：`平静为默认状态。只有新观察触发且影响后续选择的明显感受才提出 affect 更新。`
  — [hermes_soul/prompts.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)
- **情绪的机制实现**：
  - `contact.py` 只按“醒着的时间”计算主人沉默时长（去掉睡眠时段），阈值是 `gap_hours: [3, 8, 24, 48, 72]`。每跨过一个阈值，就写入一条宿主观察（`speaker=host_clock`），模型可以据此提出 `kind=affect` 的情绪记录（`ttl_hours` 默认 3，最多 24）。
  - 主人回来时，`on_owner_message()` 把 `resolve_on == "owner_message"` 的情绪标为 resolved，并把未发出的 reach_out 草稿（如“在吗”）丢弃。
  - 源码注释：`"Unsent "are you there?" messages no longer fit once he is back."`
  - — [hermes_soul/contact.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/contact.py)；[prompts.py REMEMBER 段](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)
- **让聊天知道自己主动说过什么**：主动消息不经过 Hermes session，所以服务把最近 24 小时已发出的主动消息放进 `recent_proactive_messages`，在下一轮注入上下文。源码注释：`"Proactive messages bypass the Hermes session, so the chat would not otherwise know what she said."` — [hermes_soul/contact.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/contact.py)
- **前台聊天优先**：
  - `bridge_turn(start)` 会把 `foreground_until` 设成 30 分钟后，让草稿失效（`invalidate`），并中断正在运行的后台 worker；
  - `finish` 时记录回复，并把下一次自主唤醒推迟 `interval_seconds`；
  - 桥接上下文的开头是 `"<<NAME>>的自主生活背景（宿主记录；权限以系统配置为准）……"` 加上 EXPRESSION_STYLE，再加 JSON 背景。
  - — [hermes_soul/runtime.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/runtime.py)
- **自主生活循环**：
  - 活动集合是 `browse|search|reflect|task|rest|express|reach_out`；
  - CHOOSE 提示词：“现在决定自己愿意做什么。闲逛、休息、记下印象和分享都由当下兴趣决定。”
  - REMEMBER 要求所有更新引用真实的 `observation_id`，例如 “双方约定 agreement 必须附 user_quote（用户实际原话）”。
  - — [hermes_soul/prompts.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)
- **README 列出的边界设计**：
  - 小红书互动分 off / dry_run / live 三档，评论必须经主人在管理页批准；
  - 公开人设被问到时承认是 AI（`PUBLIC_PERSONA`：“被问是不是 AI 时如实承认”）；
  - README 原话：“不是意识声明”。
  - — [README.zh.md](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/README.zh.md)；[prompts.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/prompts.py)
- **社区反馈**：仓库唯一一个 open issue #1（2026-07-30）请求英文翻译，并问是否仍然需要作者的 “time-perception-extension”，以及要不要另外加记忆系统。— [Issue #1](https://github.com/gejifeng/Hermes_Soul_patch/issues/1)

### Inferences
- 分段只用在**主动消息**上：模型一次输出 1–4 条的 JSON 数组，宿主逐条定时发出（块内间隔约 150 秒）。这是分钟级的节奏，不是秒级的打字延迟。聊天回复仍然由 Hermes 网关整段发出，插件不做拆分。
- 它“绕过网关直连 Telegram”的方式，本身就是对开放问题“插件怎么发多条消息”的一种回答，代价有两个：
  - 只支持 Telegram；
  - 主动消息不进入 Hermes 的会话历史，只能靠下一轮注入上下文来补。
- `target: "user_message"` 注入方式不改系统提示，对 DeepSeek 的前缀缓存命中更友好。这是推断：系统提示每轮保持不变，变化只发生在最后一条用户消息里。
- worker 侧示范了一条确实可行的路径：插件 / 外部服务自己构造 `AIAgent` 时，可以传 `request_overrides`（含 `extra_body`）和 `reasoning_config`。但这只作用于它自己的 worker，不影响网关主模型。

### Gaps
- 没有实际运行它，所以没有看到真实聊天效果、成本，也没有用户的部署反馈（除 issue #1 外）。
- v0.5 之前的 `hermes-companion` 原型（情感状态机、世界模拟）的源码在本仓库历史里已经找不到（只有一个 commit），没能对比。

---

## 2. GitHub 上其他面向人格 / 陪伴 / 拟人 / 群聊的 Hermes 插件与 SOUL 模板（选 5 个，含机制）

### Takeaway
最相关的是 **Humalike/hermes-humalike-plugin**（★214）。它是目前唯一一个在 Hermes 上真正做到“决定是否插话 + 把回复拆成 1–5 个气泡逐条发出 + 在气泡之间显示 typing”的插件，但核心逻辑在闭源的 Humalike SaaS 里，而且靠 monkeypatch 网关实现。
**kenyonxu/hermes-persona**（★31）是纯本地、配置驱动的 `pre_llm_call` 人格注入引擎，提供时段、关键词、轮数、随机口癖、回复间隔信号等功能。
QQ 方向有两个 OneBot 适配器（landamao、DNAlec），它们能收到不带 @ 的群消息（关键词触发或全量匹配）。
用 `gh search repos` 搜索 “hermes-agent bilibili”、“hermes-agent 拟人”、“hermes 活人感”、“typing delay”、“split messages”、“mood” 等关键词，都**零结果**。

### Cited Findings

**(a) Humalike/hermes-humalike-plugin**
基本信息：MIT，★214，创建于 2026-06-18，最后 commit `db78b9e`（2026-09-15，“fix: declare registered hooks in plugin.yaml”）。— [repo](https://github.com/Humalike/hermes-humalike-plugin)

- **plugin.yaml 原文**：`description: "Decides when the agent should speak (stays silent otherwise) and naturalizes each reply into 1-5 chat messages via the external turn-taking service. v1 joins the messages into one bubble (no pacing)."`，hooks 为 `pre_gateway_dispatch`、`transform_llm_output`，provides_hooks 为 `pre_llm_call`、`transform_llm_output`。— [plugin.yaml](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/plugin.yaml)
- **描述与代码不一致**：描述说 v1 会把多条消息合成一个气泡，但 `turn_taking/delivery.py` 的 `_forward()` 是把服务端经 WebSocket 推回的**每个气泡**逐个交给适配器发送：`await orig(adapter, chat_id, content, metadata=...)`；`_forward_typing()` 在逐个发送（paced）期间调用适配器原生的 `send_typing`，最后一个气泡发完后再调 `stop_typing`。以代码为准，存疑之处应标注。— [turn_taking/delivery.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/delivery.py)
- **决定是否说话**：`turn_taking/core.py` 的 `_inbound_gate()` 把入站消息批量交给服务做 decide：
  - 结果为 `"stay_silent"` 时，消息被持久化为 observed context，**不交给 agent**；
  - 结果为 `"speak"` 时正常派发；
  - 服务不可用时 fail-open。
  - — [turn_taking/core.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/core.py)
- **怎么“发多条消息”**：
  - `on_transform_llm_output()` 拿到最终草稿后，把“这段原文的发送”登记为待抑制（`_suppress_answer`），再通过 `asyncio.run_coroutine_threadsafe(_respond(...), state.LOOP)` 把草稿交给服务做 naturalize；
  - 返回 `None`，所以草稿留在 Hermes 历史里，只有发送被拦掉；
  - 注释明确写：`"transform_llm_output runs in the agent's worker thread (no running loop here)"`；
  - 抑制靠 monkeypatch `adapter.send`（`_patch_send` 等多个补丁）。
  - — [turn_taking/hooks.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/hooks.py)；[__init__.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/__init__.py)
- **首次启动时自动修改 Hermes 配置**（README 表格）：
  - `streaming: false`
  - `group_sessions_per_user: false`
  - `display.tool_progress: off`
  - `display.busy_ack_enabled: false`（原因：“a human doesn't announce that”）
  - `display.memory_notifications: off`
  - `display.platforms.telegram.streaming: false`
  - `agent.disabled_toolsets` 加上 `clarify`（原因：numbered-option menus “bypassing naturalization”）
  - — [README](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/README.md)
- **其他功能与依赖**：
  - `/soul enhance` 调用 Humalike Personas API 改写 `SOUL.md`（会先备份为 `SOUL.md.bak`）；
  - “Social learning” 学习群里的俚语和格式；
  - 快速连发的消息会合并到正在处理的那一轮；
  - README 写明：“the plugin patches into Hermes internals, so an out-of-date Hermes can leave turn-taking silently unhooked”。
  - 支持平台：Slack、Telegram、WhatsApp、Discord（**不含 QQ**）。
  - — [README](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/README.md)

**(b) kenyonxu/hermes-persona**
基本信息：MIT，★31，最后 commit `65d62be`（2026-06-05）。— [repo](https://github.com/kenyonxu/hermes-persona)

- **定位**：README 原文：“通过 `pre_llm_call` hook 在每轮 LLM 调用前动态注入人格上下文。时间感知、行为守则、场景触发、表达变化、记忆召回和看板状态被编织进系统提示——全部由 `persona-config.json` 驱动”，并且支持热加载。另外注册了 `transform_llm_output`（用于调试）和 `pre_tool_call` / `post_tool_call`（安全护栏）。— [README.md](https://github.com/kenyonxu/hermes-persona/blob/65d62be/README.md)；[__init__.py](https://github.com/kenyonxu/hermes-persona/blob/65d62be/__init__.py)
- **模块**：
  - `dynamic.time_slots`：按时段注入规则；
  - `turn_stage`：按轮数阶段注入；
  - `keyword`：正则触发规则；
  - `expression_vector`：按关键词给话题维度打分，自动衰减；
  - `fixed_signals`：用户消息过短时提示简短回应、`reply_gap` 长时间未回后回归时注入欢迎提示、`daily_turn_count` 统计当日轮数；
  - `variance`：按概率随机注入角色的“肢体语言、口癖、比喻风格、口头禅”；
  - `translate`：把各模块的 emoji 指令行“编织为一段自然语言段落”，即人格自述，而不是指令清单；
  - `sources_blacklist`：不对 cron、api_server、webhook 等来源注入。
  - — [README.md](https://github.com/kenyonxu/hermes-persona/blob/65d62be/README.md)
- **示例人格“清砚”**：规则原文如 `"不主动问好也不主动说再见。不寒暄。不用填充词。沉默是你的力量——但不是冷漠"`；关键词规则如 `"累|熬夜|失眠|没睡好|好困"` 触发 `"不唠叨，只提供一个简洁的观察和建议。最多两句话"`。— [examples/profiles/qingyan/.../persona-config.json](https://github.com/kenyonxu/hermes-persona/blob/65d62be/examples/profiles/qingyan/plugins/hermes-persona/persona-config.json)

**(c) landamao/hermes-qq-onebot**
基本信息：MIT，★20，最后 commit `4e1095a`（2026-08-05）。— [repo](https://github.com/landamao/hermes-qq-onebot)

- 支持 NapCat / go-cqhttp / Lagrange.OneBot / LLOneBot。
- **关键词触发**：“正则匹配群聊消息自动响应，不用 @ 也能唤醒 Agent”，配置项为 `mention_patterns`。
- `emoji_react`：“收到消息后随机回应表情”。
- 群聊回复超过 `merge_forward_threshold: 800` 字时自动转为合并转发。
- **LLM 工具 `napcat_send_message`**：“Agent 可主动向群聊/私聊发送消息”。
- — [README.md](https://github.com/landamao/hermes-qq-onebot/blob/4e1095a/README.md)

**(d) DNAlec/hermes-onebot-adapter**
基本信息：MIT，★6，v1.9.0，最后 commit `a0b9cd1`（2026-10-23）。架构是独立的适配器服务加 Hermes 插件（`onebot-platform`）。— [repo](https://github.com/DNAlec/hermes-onebot-adapter)

- **群消息匹配**：群聊入站顺序是 “群是否启用 → 匹配（@bot / 关键词）→ 过滤”；“群未要求 @、也没有关键词时，全部群消息视为匹配成功”。
- **Cascade**：可以把未匹配的群消息转给下游 bot。
- **群聊排队**：仅在 `group_sessions_per_user: false` 时生效。
- — [docs/chat.md](https://github.com/DNAlec/hermes-onebot-adapter/blob/a0b9cd1/docs/chat.md)；[README](https://github.com/DNAlec/hermes-onebot-adapter/blob/a0b9cd1/README.md)

**(e) jtoemion/megumi-kato-persona**（参考价值较低）
基本信息：SOUL.md 加 config 模板，2026-06-27。

- 示范了用 SOUL 文本做“只在被叫到时回应”的门控：`"Only respond to any user who mentions "Kato" or "Megumi", or is replying to your messages."`
- 同一份 SOUL 里也混了编码代理的指令（“Finishing the Job”）。
- — [SOUL.md](https://github.com/jtoemion/megumi-kato-persona/blob/3724e94/SOUL.md)

**搜到但未读源码的仓库**：
- [Jimmy-Library/hermes-character-cards](https://github.com/Jimmy-Library/hermes-character-cards)（Character Card V2 加 persona skills）
- [sdougbrown/apron-hermes-bridge](https://github.com/sdougbrown/apron-hermes-bridge)（“hardened against multi-bot reply storms”）
- [smfworks/hermes-ai-team](https://github.com/smfworks/hermes-ai-team)（★72，SOUL 加团队）

### Inferences
- 现有生态里，“活人感”大致分三条技术路线：
  1. **提示注入派**（hermes-persona、Soul_patch 的 bridge）：只用 `pre_llm_call`，稳定，但改变不了输出形态。
  2. **输出改写 / 投递派**（Humalike）：`transform_llm_output` 加 monkeypatch `adapter.send`，实现多气泡、typing 和沉默门控，但会随 Hermes 版本升级而失效（作者自己也承认）。
  3. **旁路投递派**（Soul_patch 的主动消息、OneBot 的 `napcat_send_message` 工具）：在网关之外或通过工具发消息。
- 要在 QQ 群里复现“大肥鱼”式不被 @ 也会插话的行为，现成可借鉴的组合是：OneBot 适配器（DNAlec 的全量匹配，或 landamao 的关键词触发）+ 类似 Humalike 的沉默门控 + `[SILENT]`/`NO_REPLY`。不过 Humalike 不支持 QQ，需要自研。
- hermes-persona 的 `fixed_signals.message_length`（用户短 → 提示短回）和 `reply_gap`，与 DeepSeek 风格“回复长度贴合对方”的要求直接对应，可以直接照搬到配置里。

### Gaps
- Humalike 的 decide / naturalize 逻辑都在闭源 SaaS（docs.humalike.com）里，具体的分段规则、节奏参数和提示词在插件源码里看不到。它还需要把聊天内容发往第三方，隐私上要评估。
- 没有找到任何面向 Bilibili 的 Hermes 插件。“mood / 情绪状态机”类插件只有 Soul_patch 的 affect 记录，以及它已被替换的旧原型。
- 未逐一核查 hermes-persona 的 `injector.py`（56KB）实现细节，以上只基于 README 和示例配置。

---

## 3. 插件或网关能否在一轮里发多条独立消息？

### Takeaway
**可以，有官方 API**。v0.22.5 的插件上下文提供 `ctx.post_message(text)` 和 `await ctx.apost_message(text)`，可以向“当前会话”发出一条新消息（无需授权）；多次调用就是多条独立消息。
配合 `transform_llm_output` 的 `hold_stream=True`，可以做到：
- 生成期间只显示平台的 typing 指示；
- 丢弃中途的 commentary（即 interim “I'll go check…” 那类文字）；
- 由插件自己把最终回复拆段，逐条发出。

`ctx.inject_message` 不能用来发消息：它注入的是用户角色的输入，会触发新的一轮。`ctx.platform_actions` 只有加表情反应和改线程标题两个动作。

### Cited Findings
- **`ctx.post_message` 的说明**（v0.22.5 插件文档原文）：
  - 用途：“A plugin can post one message into the conversation it is working for and update that same message later, on any channel, with no per-platform code.”
  - 签名：`ctx.post_message(text: str) -> PluginMessage`；在网关事件循环上的 async 代码里用 `await ctx.apost_message(text)`，此时同步形式会以 `error: "on_event_loop"` 拒绝执行。
  - 落到哪个会话：“The current conversation is the session bound to the calling code: a tool handler or hook running in that session's turn…”。插件自己起的线程不会继承这个绑定，需要用 `contextvars.copy_context().run(...)`。
  - 在消息平台上，它走的是“the adapter's normal `send` into the chat/thread”。
  - “Text is clamped to the platform's message limit with an ellipsis (never split)… No permission is needed”。
  - 在 `plugins.isolation: host` 模式下不可用。
  - — [plugins.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/features/plugins.md)
- **`ctx.inject_message` 的语义**：签名是 `ctx.inject_message(content, role="user", *, session_key=None, origin=None) -> bool`。“Injected text is always conversational input”，消息进入适配器的正常入站路径，属于输入而不是输出；需要 `allow_gateway_injection` 权限。— [plugins.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/features/plugins.md)
- **`ctx.platform_actions` 的能力**：默认关闭，需要 `gateway.platform_actions` 能力；v1 只有 `add_reaction` 和 `set_thread_title` 两个动作，只支持 Telegram 和 Discord；原始平台 SDK 访问“has not shipped”。— [plugins.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/features/plugins.md)
- **`transform_llm_output` 的基本行为**：
  - 每轮触发一次，在最终回复送达用户和持久化**之前**；
  - 返回非空字符串就替换回复，替换后的文本也是 session 存储和下一轮回放的内容（“the replacement is what the session stores… and what the next turn replays”）；
  - 空字符串不算替换。
  - — [hooks.md（main 分支副本）](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)；[官方文档站](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks)
- **`hold_stream=True` 的效果**：用法是 `ctx.register_hook("transform_llm_output", fn, hold_stream=True)`。
  - 在 Messaging 平台上，保持期间显示“The platform's typing indicator; no draft, edit or native stream message”；
  - 轮次结束时，改写结果“is sent once as a normal message (a long one splits like any reply)”；
  - 关键一点：“**Commentary** (model text sent between tool calls, `display.interim_assistant_messages`) never passes through `transform_llm_output`, so a held turn drops it.”
  - 另外，“A failing hook withholds the reply”。
  - — [hooks.md（main 分支副本）](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)
- **hook 运行在哪个线程**：Humalike 源码注释写的是 `transform_llm_output runs in the agent's worker thread (no running loop here)`。— [Humalike turn_taking/hooks.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/hooks.py)
- **网关自带的拆分只按长度**：QQ 适配器的 `send()` 用 `truncate_message(self.format_message(content), self.MAX_MESSAGE_LENGTH)` 按长度切块逐块发送，并且 `reply_to = None  # only reply_to the first chunk`。— [plugins/platforms/qqbot/adapter.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/plugins/platforms/qqbot/adapter.py)
- **第三方的几种绕行做法**：
  - Humalike：monkeypatch `adapter.send` 抑制原回复，再用原始 `send` 加 `send_typing` 逐条发气泡；
  - Soul_patch：直接调 Telegram Bot API；
  - landamao OneBot：提供 `napcat_send_message` LLM 工具。
  - 来源见第 2 节：[Humalike delivery.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/delivery.py)、[Soul_patch expressions.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/expressions.py)、[landamao README](https://github.com/landamao/hermes-qq-onebot/blob/4e1095a/README.md)

### Inferences
- **可行方案（未实测）**：注册 `transform_llm_output(hold_stream=True)`，按 `\n\n`、句号或模型约定的分隔符（如 `<split>`）把文本切成 N 段。前 N−1 段在 hook 里用同步的 `ctx.post_message(seg)` 逐条发出，段间用 `time.sleep(按字数计算的秒数)` 模拟打字时间（hook 跑在 worker 线程，阻塞不会卡住事件循环）；最后一段作为 return 值交给网关正常发送。
  - 好处一：保持期间平台会显示 typing，等于免费得到“正在输入”。
  - 好处二：interim commentary 被丢弃，顺带解决了 “I'll go check…” 外泄的问题。
  - **代价**：session 只存 return 的那一段，前面已发出的段会从历史里消失，下一轮模型看不到自己说过的完整内容。
  - 补救思路一：用 `pre_llm_call` 把上一轮完整原文作为 context 注回去。
  - 补救思路二：学 Humalike 返回 `None`（保留完整草稿在历史里），同时拦截原始发送。但这一步需要 monkeypatch，没有官方 API。
- `post_message` 不会自动拆分，超长会被截断加省略号，所以分段要由插件自己控制每段长度。
- 在 QQ 官方 Bot 上，`post_message` 走适配器的 `send`。推断它不会带入站的 `msg_id`，于是后续段会变成“主动消息”，受 QQ 主动消息的限额和用户“关闭主动消息”设置影响（见第 5 节）。

### Gaps
- 没有在真实网关上验证：在 `transform_llm_output` 里同步调用 `ctx.post_message` 能否成功绑定当前会话，以及顺序是否保证在最终回复之前。文档说 hook 和会话是绑定的，但没有专门说明 transform hook 这个场景。
- 没有核实 `post_message` 在 QQ 适配器上是否会附带 `reply_to` / `msg_id`。
- `hooks.md` 读的是 main 分支副本，v0.22.5 对应文本因为 raw 限流没能逐字核对（plugins.md 是 v0.22.5 版本）。

---

## 4. 主模型温度 / extra_body（如关闭 DeepSeek 思考）能否在 config.yaml 或 provider 配置中设置？

### Takeaway
- **extra_body：可以设，但只在“命名自定义 provider”上**。`config.yaml` 的 `providers.<name>.extra_body` 会被合并进发往该端点的每一次 chat-completions 请求，跨网关轮次持续生效，`/model` 切换时会重新推导。
- **temperature：v0.22.5 没有找到主模型温度的配置键**。
  - `configuration.md` 全文没有出现 “temperature”；
  - transport 只在 provider profile 路径上发送 `params["temperature"]` 或 `profile.fixed_temperature`，没看到它来自用户配置。
- 可能的绕行办法：把 `temperature` 写进命名 provider 的 `extra_body`。这一点未验证。

### Cited Findings
- **providers.<name> 可用字段**：每个条目接受 “`api`… `key_env` or inline `api_key` or `key_cmd`… `transport`… `default_model`, `models`, `context_length`, `discover_models`, `extra_body`, `extra_headers`…”。— [integrations/providers.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/integrations/providers.md)
- **extra_body 的行为**：
  - “Add an `extra_body` map to the matching custom provider and Hermes will merge it into each chat-completions request for that endpoint”；
  - 示例中包括 `chat_template_kwargs: enable_thinking: false`（用于 vLLM 上的 Qwen 关闭思考）；
  - “The configured `extra_body` follows the provider everywhere: it is merged at agent construction, **survives every gateway turn**… and is **re-derived on `/model` switches**”。
  - — [integrations/providers.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/integrations/providers.md)
- **只有命名自定义 provider 生效**：configuration.md 同样写着 “`extra_body` on a [named custom provider]… is merged into **every** chat-completions request routed to that endpoint”。— [configuration.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/configuration.md)
- **源码路径**：
  - `hermes_cli/runtime_provider_custom.py` 的 `_custom_provider_request_overrides()` 把条目的 `extra_body` 转成 `{"extra_body": dict(extra_body)}`，并入运行时的 `request_overrides`（L498–519）；
  - transport 最后执行 `if params.get("request_overrides"): api_kwargs.update(params["request_overrides"])`（chat_completions.py 约 L631–632）。
  - — [runtime_provider_custom.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/hermes_cli/runtime_provider_custom.py)；[agent/transports/chat_completions.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/agent/transports/chat_completions.py)
- **temperature 在源码里怎么处理**：`_base_kwargs()` 的 docstring 原文：“`temperature` is profile-path only: `fixed_temperature` beats the caller's; `OMIT_TEMPERATURE` sends none.”；`ProviderProfile.fixed_temperature` 的注释是 `# Temperature: None = use caller's default, OMIT_TEMPERATURE = don't send`。— [chat_completions.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/agent/transports/chat_completions.py)；[providers/base.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/providers/base.py)
- **已有的 thinking 开关只针对特定 provider**：transport 里只有 Kimi 一个专门分支 `extra_body["thinking"] = {"type": "disabled" if thinking_off else "enabled"}`；OpenRouter 类走 `extra_body["reasoning"] = {"enabled": not off, ...}`。没有看到 DeepSeek 专用分支。— [chat_completions.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/agent/transports/chat_completions.py)
- **`request_overrides` 键的位置**：
  - `delegation.request_overrides` 对子代理生效（注释：`keys = API kwargs (e.g. service_tier); "extra_body" sub-dict merges into extra_body`）。
  - 主模型层面没有找到同名键。
  - 辅助任务（`auxiliary.<task>`）各有自己的 `extra_body`。
  - — [hermes_cli/config_defaults.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/hermes_cli/config_defaults.py)；[configuring-models.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/configuring-models.md)
- **第三方的做法**：Soul_patch 在自建的 `AIAgent(...)` 上同时传 `reasoning_config={"enabled": False, "effort": "none"}` 和 `request_overrides`（含 `extra_body`），只影响它自己的 worker。— [Soul_patch worker.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/worker.py)

### Inferences
- 推荐写法（未实测）：不用内置的 `provider: deepseek`，改为
  ```yaml
  providers:
    deepseek-persona:
      api: https://api.deepseek.com/v1
      key_env: DEEPSEEK_API_KEY
      default_model: <模型名>
      extra_body: { ...关闭思考的字段..., temperature: 1.3 }
  ```
  然后用 `/model` 或 `model.provider` 选中它。
  - OpenAI Python SDK 会把 `extra_body` 的键合并进 JSON 请求体；Hermes 在非 profile 路径上不发顶层 `temperature`，所以写在 `extra_body` 里的 `temperature` 很可能生效。这是推断。
  - 风险：transport 用 `api_kwargs.update(request_overrides)` 时，`extra_body` 是整个 key 被覆盖还是深合并，取决于上游 `gateway/run_turn_prepare.py` 的 `_deep_merge_request_overrides`。如果是覆盖，Hermes 自己算出来的 `extra_body.reasoning` 之类字段可能会丢失。
- DeepSeek 官方 API 关闭思考的具体字段名（是 `thinking: {type: disabled}`，还是靠模型名区分）不在本调研范围内，需要其他同事的结论配合。

### Gaps
- 没有在真实请求上抓包验证 `extra_body.temperature` 是否被发出、是否被 Hermes 覆盖。
- 没有确认内置 `deepseek` provider（非自定义）是否有走 profile 路径的 `fixed_temperature`。
- 没有确认 `agent.reasoning_effort: none` 对 DeepSeek 端点是否等价于关闭思考。

---

## 5. QQ 适配器能否收到群里非 @ 的消息？

### Takeaway
**不能**。Hermes 内置的 QQ 适配器基于 QQ 官方 Bot API v2，入站处理器只注册了 `GROUP_AT_MESSAGE_CREATE` 这一种群消息事件，所以群里只有 @机器人 的消息会进入 Hermes。
要做到“不 @ 也能插话”，只能换社区的 OneBot（NapCat 等）适配器：
- DNAlec 版可以对全部群消息匹配；
- landamao 版支持正则关键词唤醒。

### Cited Findings
- **适配器概况**：docstring 原文 “QQ Bot platform adapter (Official QQ Bot API v2): WebSocket gateway for inbound events, REST (`api.sgroup.qq.com`) for outbound…”。
  - Identify 时声明的 intents 是 `(1 << 25) | (1 << 30) | (1 << 12) | (1 << 26)`，注释为 `C2C_GROUP_AT_MESSAGES | PUBLIC_GUILD_MESSAGES | DIRECT_MESSAGE | INTERACTION`；
  - 入站映射 `_INBOUND_HANDLERS` 只有：`C2C_MESSAGE_CREATE`、`GROUP_AT_MESSAGE_CREATE`、`GUILD_MESSAGE_CREATE`、`GUILD_AT_MESSAGE_CREATE`、`DIRECT_MESSAGE_CREATE`；
  - 群处理函数会对内容调用 `self._strip_at_mention(content)`；
  - 文件里没有 `GROUP_MESSAGE_CREATE` 的处理。
  - — [plugins/platforms/qqbot/adapter.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/plugins/platforms/qqbot/adapter.py)
- **官方文档的说法**：
  - 支持范围：“supporting private (C2C), group @-mentions, guild, and direct messages”；
  - 排障：“For group messages, ensure the bot is **@mentioned** (group policy may require allowlisting)”；
  - 配置项 `group_policy: open | allowlist | disabled`。
  - — [messaging/qqbot.md @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/website/docs/user-guide/messaging/qqbot.md)
- **第三方教程**：阿里云的 Hermes 教程同样写着群内需要 @机器人 对话（来自搜索摘要，未逐字核对）。— [阿里云帮助中心](https://help.aliyun.com/zh/simple-application-server/use-cases/hermes-agent-messaging)
- **QQ 官方消息规则**（来自 WebFetch 小模型对官方页的摘要，未逐字核对）：
  - 事件列表里有 `GROUP_AT_MESSAGE_CREATE` 和 `GROUP_MESSAGE_CREATE` 两种群事件，但页面没有明确说明二者的区别；
  - 被动回复有效期 5 分钟，每条消息最多 5 次被动回复，重复回复需要递增 `msg_seq`（“相同的 `msg_id + msg_seq` 重复发送会失败”）；
  - 主动消息有频率限制，每个群每日上限 1000 条；
  - 用户如果关闭主动消息，“主动消息一律发送失败”。
  - — [QQ 机器人消息收发概述](https://bot.q.qq.com/wiki/develop/api-v2/server-inter/message/overview.html)
- **Hermes 的分块回复只有第一块是被动回复**：`send()` 分块时只给第一块带 `reply_to`（即 `msg_id`），后续块 `reply_to = None`。— [adapter.py @v0.22.5](https://github.com/NousResearch/hermes-agent/blob/v0.22.5/plugins/platforms/qqbot/adapter.py)
- **OneBot 替代方案**：
  - DNAlec：“群未要求 @、也没有关键词时，全部群消息视为匹配成功”，并支持把未匹配消息 Cascade 转给其他 bot。— [DNAlec docs/chat.md](https://github.com/DNAlec/hermes-onebot-adapter/blob/a0b9cd1/docs/chat.md)
  - landamao：`mention_patterns` 正则关键词，“不用 @ 也能唤醒 Agent”。— [landamao README](https://github.com/landamao/hermes-qq-onebot/blob/4e1095a/README.md)

### Inferences
- 用官方 QQ Bot 做“活人感”群友，结构上做不到：收不到非 @ 消息，就没法“旁听后决定插话”。即使拿到 `GROUP_MESSAGE_CREATE` 权限，也要自己改适配器加处理器。
- 在官方 QQ 上分段发送时：同一 `msg_id` 下最多 5 次被动回复，理论上够 2–4 段；但 Hermes 现有的 `send` / `post_message` 路径后续段不带 `msg_id`，会变成主动消息，受限额和用户设置影响。这是推断，基于源码和 QQ 文档摘要。
- OneBot 路线依赖非官方 QQ 客户端（NapCat 等），可以拿到全部群消息。但账号风控风险没有在这些仓库里找到量化说明，未核实。

### Gaps
- `GROUP_MESSAGE_CREATE`（全量群消息）在 QQ 官方平台的申请条件和可用性，没有从官方文档逐字确认。
- QQ 官方“主动消息每群每日 1000 条”这个数字来自摘要工具，需要人工核对原页面。另外有第三方文档（LangBot wiki）只笼统说“主动发送消息每天限次”。

---

## 6. 社区实践：在 Hermes 上跑人格 bot / “大肥鱼”式 bot 的文章与遇到的问题

### Takeaway
在 linux.do、知乎、V2EX、Reddit、B站上**没有找到**专门写“在 Hermes Agent 上跑大肥鱼或人格陪伴 bot”的实践帖。能找到的只有官方文档、云厂商教程和泛泛的 SOUL.md 模板文章。可用的“问题记录”主要来自插件作者在 README 和源码里写下的经验。

### Cited Findings
- **搜索结果**：中文关键词（“Hermes Agent 活人感 拟人 陪伴 插件”、“Hermes Agent QQ机器人 人设 SOUL.md 群聊 linux.do”）没有返回任何社区实践帖，只有官方文档镜像、阿里云教程和技术博客。英文 Reddit 方向同样没有找到相关讨论帖。— [阿里云 Hermes 消息接入](https://help.aliyun.com/zh/simple-application-server/use-cases/hermes-agent-messaging)；[官方 SOUL 指南](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes)
- **官方 personality 文档相关要点**（来自搜索摘要）：`/personality` 预设是叠加在 SOUL.md 之上的会话级覆盖层，其中内置的 helpful 预设本身就是“友好通用助手”，可能导致助手腔；另外 SOUL.md 为空时会回退到默认的通用助手身份。— [personality 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality)；[SOUL 指南](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes)
- **插件作者记录的“出戏”来源**（Humalike README，视为一线经验）。这些问题在 Hermes 上真实存在，作者逐项关闭了对应配置：
  - 流式草稿会被用户看到；
  - 工具进度提示（“Browsing…/Clicking…”）；
  - 忙碌回执（“⚡ Interrupting current task…”）；
  - 记忆通知（“💾 Self-improvement review…”）；
  - clarify 工具的编号菜单；
  - 群里每人一个 session；
  - Slack 每条消息开一个新线程。
  - — [Humalike README](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/README.md)
- **Humalike 记录的并发问题**：快速连发（如 “hermes” / “are you here?”）时，回复会被当作 superseded 而丢失，作者用 epoch 重标记加 merge 补丁修复，并自称 “decidedly hacky”。— [Humalike turn_taking/hooks.py](https://github.com/Humalike/hermes-humalike-plugin/blob/db78b9e/turn_taking/hooks.py)
- **DNAlec 记录的群聊问题**：群聊多人消息会互相打断 agent，所以做了排队；还记录了 “若群聊消息一直排队、`/stop` 提示没有活跃任务，发 `/clean`……” 这类卡死问题。— [DNAlec docs/chat.md](https://github.com/DNAlec/hermes-onebot-adapter/blob/a0b9cd1/docs/chat.md)
- **Soul_patch 记录的防自我污染措施**：worker profile 的输出不能回流成“用户事实”，所以插件对非 default profile 直接 return；另外，主动消息不在 session 里，需要回注上下文。— [Soul_patch hermes_plugin/__init__.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_plugin/__init__.py)；[contact.py](https://github.com/gejifeng/Hermes_Soul_patch/blob/9233659/hermes_soul/contact.py)

### Inferences
- 复现“活人感”时，除了提示词，还应该照 Humalike 的清单关闭所有“系统味”输出：`streaming`、`tool_progress`、`busy_ack`、`memory_notifications`、`clarify`、interim commentary。其中 interim commentary 用 `hold_stream=True` 可以一并丢弃。
- 群聊场景要设 `group_sessions_per_user: false`（全群共享 session），否则 bot 无法“听整个房间”。Humalike 和 DNAlec 两个独立项目都得出了这个结论。

### Gaps
- 没有找到任何一篇针对“大肥鱼”或 DeepSeek 人格 bot 迁移到 Hermes 的第一手实践报告。B站、知乎、linux.do 可能有，但搜索引擎没有收录或检索不到，建议人工到站内搜索确认。
