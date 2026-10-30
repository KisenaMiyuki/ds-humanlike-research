# 目标环境与可移植层：在 Hermes Agent（及其他通用 harness）上稳定复现 DeepSeek「活人感」

> 研究日期：2026-10-30。Hermes Agent 源码与文档通过 `gh api repos/NousResearch/hermes-agent/contents/...` 读取的是 main 分支当日版本（最新 release v0.22.5，2026-10-23）。DeepSeek 文档通过 curl 抓取 api-docs.deepseek.com 当日页面。
> 说明：下文 GitHub 链接统一写成 `blob/main/...` 形式，即我通过 API 读到的文件路径；官方文档站的对应页面在 `https://hermes-agent.nousresearch.com/docs/...`。

---

## 问题 1：「Hermes」到底指什么？人设/风格在哪里配置？它自带的系统提示词会不会和自定义人设打架？

### Takeaway
「Hermes」几乎可以确定是 Nous Research 的开源 **Hermes Agent**（github.com/NousResearch/hermes-agent）。DeepSeek 官方 API 文档把它列为正式的「Agent Integrations」之一，和 DeepSeek Harness、Claude Code、OpenClaw 并列。人设主要写在 `~/.hermes/SOUL.md`：它占系统提示词第 1 槽，并整体替换内置的默认身份。但 SOUL.md 之后 Hermes 还会追加一串偏「干活助手」的指导块。其中「工具使用强制」和「执行纪律」这两块按模型名自动注入，**DeepSeek 默认就在注入名单里**。网关还默认把中途的 assistant 旁白发进聊天。这些是跨 harness 迁移时「助手腔」回流的主要来源，不过都能通过配置关掉。

### Cited Findings

**身份识别**
- DeepSeek API 文档左侧导航「Agent Integrations」列出：DeepSeek Harness、Claude Code、Codex、OpenCode、OpenClaw、**Hermes**、Reasonix、WorkBuddy/CodeBuddy、Qoder。其中 Hermes 页面写的是「Hermes is an open-source AI agent built by Nous Research」，安装用 `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`，然后执行 `hermes setup` → Quick Setup → 选 DeepSeek → Base URL 填 `https://api.deepseek.com` → 选 **`deepseek-v4-pro`** — [DeepSeek API Docs: Integrate with Hermes Agent](https://api-docs.deepseek.com/quick_start/agent_integrations/hermes)
- 仓库描述为 "The agent that grows with you"。截至 2026-10-30，GitHub API 显示约 258,593 stars，最近一次 release 是 v0.22.5（2026-10-23），v0.22.x 在 10 月中下旬连发了 5 个补丁版本 — [GitHub API: NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)；[Releases](https://github.com/NousResearch/hermes-agent/releases)
- 另一个可能的指代是 Nous 的 **Hermes 模型家族**（Hermes 4 / 4.3）。Hermes 4 405B 于 2025-08-26 发布，使用 Llama-3-Chat 格式，是混合推理模型（可输出 `<think>…</think>`，也可直接回答），宣称训练目标是 steerability、低拒答、中立且由用户主导的行为。Hermes 4.3 36B 的模型卡用 "You are Hermes 4. Be concise and helpful." 作为 system 示例，tags 里有 roleplaying — [HF: Hermes-4.3-36B](https://huggingface.co/NousResearch/Hermes-4.3-36B)；[OpenRouter: hermes-4-405b](https://openrouter.ai/nousresearch/hermes-4-405b)；[HF: Hermes-4-70B 镜像卡](https://huggingface.co/N0rv3ll/Hermes-4-70B)（这些多为镜像/聚合页，技术报告 arXiv 2508.18255 未打开）

**SOUL.md：人设主入口**
- `SOUL.md` 是「primary identity」，占系统提示词 **slot #1**，并**替换**硬编码的默认身份。它只从 `HERMES_HOME`（默认 `~/.hermes/SOUL.md`）加载，不读当前工作目录。不存在时会自动生成 starter 文件，已有文件永不覆盖。文件为空或读不到时回退到内置默认身份。内容经过 prompt-injection 扫描和超长截断后**原样注入，不加任何包裹语** — [Hermes docs: Personality & SOUL.md（源文件 website/docs/user-guide/features/personality.md）](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)；[文档站中文版](https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/features/personality)
- 官方建议 SOUL.md 写语气、沟通风格、直接程度、「what to avoid stylistically」、如何处理不确定和分歧。项目规范、路径、命令放进 `AGENTS.md`。原话是「if it should follow you everywhere, it belongs in SOUL.md」 — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)
- 内置默认身份（`DEFAULT_AGENT_IDENTITY`，与仓库根目录 SOUL.md 相同）："You are Hermes Agent, built by Nous Research. Be direct: match the length of your reply to the weight of the ask … No filler ("Great question," "I'd be happy to") … Depth is earned …" — [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)；[SOUL.md（repo 根）](https://github.com/NousResearch/hermes-agent/blob/main/SOUL.md)
- `/personality <name>` 是**会话级叠加层**。内置 14 个预设（helpful/concise/technical/creative/teacher/kawaii/catgirl/pirate/shakespeare/surfer/noir/uwu/philosopher/hype），也可以在 `config.yaml` 的 `agent.personalities` 里自定义。另有 `agent.system_prompt` 用于手写系统提示，只在没选 personality 时生效 — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)
- 多人设/多实例：每个 profile 有自己的 HERMES_HOME，因此有独立的 SOUL.md（`hermes profile create --clone`）。社区建议 SOUL.md 写 3–5 句，修改**在新会话才生效**，因为系统提示词在会话开始时构建并缓存 — [journal: How to Play with SOUL.md in Hermes Agent（2026-05-13）](https://journal-ex2.pages.dev/posts/tech/260513-hermes-persona-soul)；源码 docstring 也写着 "Built once per session and reused across turns (only context compression triggers a rebuild) so the upstream prefix cache stays warm" — [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
- 第三方 Soul Spec（ClawSouls）的人设包装进 Hermes 时，会把 IDENTITY.md/STYLE.md 合并进 `~/.hermes/SOUL.md`，因为 Hermes 不支持单独的 identity 文件 — [ClawSouls: Hermes Agent + Soul Spec](https://docs.clawsouls.ai/docs/guides/hermes)（来自搜索摘要，未全文核实）
- Bot Mode（桌面端）支持多个 Bot，各自带头像、语音、Routines、群聊、**Bot-to-bot messaging**、跨机器 Bot 主动发起私信（`hermes peer`） — [website/docs/user-guide/bot-mode.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/bot-mode.md)（只看了目录结构）

**系统提示词实际装配顺序（决定「助手腔」从哪来）**
- 官方概述的提示词栈：1) SOUL.md → 2) tool-aware behavior guidance → 3) memory/user context → 4) skills guidance → 5) context files（AGENTS.md、.cursorrules）→ 6) timestamp → 7) platform-specific formatting hints → 8) `/personality` 等叠加层 — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)
- 源码中的三层缓存结构：**stable**（SOUL/默认身份、Hermes 自助说明 `HERMES_AGENT_HELP_GUIDANCE`、TASK_COMPLETION / PARALLEL_TOOL_CALL guidance、工具指导、STEER_CHANNEL_NOTE、模型门控的 TOOL_USE_ENFORCEMENT / EXECUTION_GUIDANCE、pinned skills、coding brief、平台提示）→ **context**（调用方 system_message、AGENTS.md 等上下文文件、工作区快照）→ **volatile**（skills 索引、MEMORY/USER 块、外部记忆 provider、插件段落 "after_memory"、profile 行、时间戳、运行环境提示） — [agent/system_prompt.py `build_system_prompt_parts`](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
- `TOOL_USE_ENFORCEMENT_GUIDANCE` 原文含 "You MUST use your tools to take action … Every response should either (a) contain tool calls that make progress, or (b) deliver a final result to the user." 它按模型名子串自动注入，名单为 `("gpt","codex","gemini","gemma","grok","glm","qwen","deepseek","muse")`。另一个独立门控 `EXECUTION_GUIDANCE_MODELS` 注释说明 deepseek/kimi/qwen/glm/minimax 等也会拿到 OPENAI_MODEL_EXECUTION_GUIDANCE，并注明 "Claude does not exhibit these modes" — [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
- 两个门控都可配置："agent.tool_use_enforcement / agent.execution_guidance: "auto" (default) matches the hardcoded model lists; true/false force; a list gives custom model-name substrings"。TASK_COMPLETION / PARALLEL_TOOL_CALL guidance 也各有开关（"each gated by its config.yaml key"），且只在有工具时注入 — [agent/system_prompt.py `_guidance_parts`](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
- 平台提示：`cron` 提示为 "There is no user present … Your final response is automatically delivered…"；`api_server` 提示为 "assume plain text. No markdown … Treat this like a conversation, not a document. Keep responses brief and natural."。各消息平台的 hint 随该平台插件分发（`PlatformEntry.platform_hint`） — [agent/prompt_builder.py `PLATFORM_HINTS`](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
- 子代理/委派上下文：设置了 `skip_context_files` 时会回退到默认身份。cron 则保留人设（源码注释："SOUL.md (primary identity; cron keeps the persona while skipping cwd instructions…)"） — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)；[agent/system_prompt.py `_identity_parts`](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
- 上下文文件截断：默认 `CONTEXT_FILE_MAX_CHARS = 20_000`，按「头 70% / 尾 20%」截断，可用 `context_file_max_chars` 覆盖 — [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)

**记忆、技能、定时任务、网关、模型提供商（逐项文档位置）**
- 记忆：`MEMORY.md`（agent 自己的笔记，2,200 字符 ≈ 800 tokens）+ `USER.md`（用户画像，1,375 字符 ≈ 500 tokens），位于 `~/.hermes/memories/`，以带用量百分比的冻结块注入系统提示。**不会自动压缩**，写满时 memory 工具报错，要求合并或替换。官方建议在网关场景用 `/new` 制造会话边界，让记忆生效 — [features/memory.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/memory.md)
- 记忆指导原文要求条目写成陈述事实而非指令："'User prefers concise responses' ✓ — 'Always respond concisely' ✗ (imperative phrasing gets re-read as a directive…)" — [agent/prompt_builder.py `build_memory_guidance`](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
- 外部记忆 provider：Honcho（用户建模）、OpenViking、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Supermemory、Memori，附有对比表和自建 provider 指南 — [features/memory-providers.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/memory-providers.md)；[features/honcho.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/honcho.md)
- Skills：按需加载的 `SKILL.md`（progressive disclosure），兼容 agentskills.io 标准，支持 `/learn` 从资料生成 skill — [features/skills.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md)
- Cron：例如 `hermes cron create "every 1h" "Summarize new feed items" --skill blogwatcher`。gateway 每 60 秒 tick 一次，每次在**隔离的新会话**中执行，最终回复自动投递到 `deliver:` 目标（如 Telegram home channel） — [features/cron.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/cron.md)
- Heartbeat：`/heartbeat every 15m <prompt>` 在**同一会话**空闲时以普通 user 消息注入（"Cache-safe … No system-prompt mutation"）。网关里可以主动唤醒，并允许以 `NO_REPLY`/`[SILENT]` 静默结束。还有防编造工作的 guard — [features/heartbeat.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/heartbeat.md)
- 消息平台（docs/messaging 目录）：Telegram、Discord、Slack、Signal、WhatsApp、Matrix、Email、SMS、**qqbot**、**weixin**、**wecom**、**yuanbao**、**dingtalk**、**feishu**、LINE、Teams、IRC、Mattermost、webhooks、Open WebUI 等 — [website/docs/user-guide/messaging/](https://github.com/NousResearch/hermes-agent/tree/main/website/docs/user-guide/messaging)
- QQ 走官方 QQ Bot API v2（私聊 C2C、群 @、频道、私信，支持语音转写）。群消息需 **@机器人** 才能收到，`group_policy` 可选 open / allowlist / disabled — [messaging/qqbot.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/qqbot.md)
- 模型提供商：DeepSeek 原生 provider（`DEEPSEEK_API_KEY`，provider id `deepseek`，base_url `https://api.deepseek.com/v1`），也能通过 Nous Portal、OpenRouter、Bedrock、Together 等使用 DeepSeek 权重 — [integrations/providers.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/integrations/providers.md)；[plugins/model-providers/deepseek/__init__.py](https://github.com/NousResearch/hermes-agent/blob/main/plugins/model-providers/deepseek/__init__.py)
- Nous Portal 被官方称为 "the recommended way to run Hermes Agent"，一次 OAuth 可用 300+ 模型（Claude、GPT、Gemini、DeepSeek、Qwen、Kimi 等） — [integrations/providers.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/integrations/providers.md)

**网关默认行为里的「助手腔」泄漏点**
- Telegram 的默认值：`tool_progress` 关、`busy_ack_detail` 关，但 **`interim_assistant_messages` 开**（"real mid-turn assistant commentary (the model literally telling you what it's about to do)"），`long_running_notifications` 也开（"⏳ Working — N min" 气泡）。可按平台在 `display.platforms.<platform>` 下关闭 — [messaging/index.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/index.md)
- 压缩进度提示默认不发到聊天平台（`compression.progress_notices: false`），失败提示总是可见 — [user-guide/configuration.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)

### Inferences
- SOUL.md 本身**不会**被 Hermes 的默认身份覆盖，因为它会整体替换 slot #1。真正和人设「打架」的是 slot #1 之后的通用指导块：工具强制、执行纪律、任务完成、并行工具、Hermes 自助说明、coding brief、平台 hint。对 DeepSeek 而言，TOOL_USE_ENFORCEMENT 中「每条回复要么调用工具要么交付最终结果」的措辞会把模型推向「任务汇报」模式。纯聊天/评论用的 persona profile 建议设 `agent.tool_use_enforcement: false`、`agent.execution_guidance: false`，并把 tools 精简到必需的几个。
- 这些指导块位于 SOUL.md 之后、同处 system 中，属于「后写者更近」。按 SillyTavern 的经验，越靠近末尾影响越大（见问题 4），所以人设最好再在末尾（user 消息侧）补一个短提醒，而不是只依赖 slot #1。
- Hermes 原生覆盖了 persona 文件、记忆/用户画像、定时主动、静默 token、群聊旁听、QQ/微信/Telegram 网关，正好是「鱼灾」类 bot 的骨架。缺的主要是「拟人化投递层」（分条、延迟）和「状态层」（情绪/好感度），这两块要靠插件补（见问题 2）。

### Gaps
- 没有逐行核实 SOUL.md 的截断上限是否与上下文文件共用 20,000 字符的动态上限，文档只说「truncated if too large」。
- TASK_COMPLETION_GUIDANCE、PARALLEL_TOOL_CALL_GUIDANCE、STEER_CHANNEL_NOTE 的原文和对应的 config 键名没有逐一读取。
- 「Hermes」在个别中文社区语境里也可能指别的项目（例如某些 QQ bot 框架同名项目），但 DeepSeek 官方集成页明确指向 Nous 的 Hermes Agent。没有找到同名的竞争性候选在 2026 年活人感社区中流行的证据。

---

## 问题 2：哪些活人感组件是纯 prompt（到处可移植），哪些需要运行时代码？如何映射到 Hermes 的具体挂点？

### Takeaway
人设、风格/反 AI 腔规则、few-shot、日记写作的「内容」都是**纯 prompt**，可以放进 SOUL.md、skill 或 pre_llm_call 注入，跨 harness 可直接搬。「是否说话/静默」「主动发消息」「长期记忆」在 Hermes 里有**原生机制**（静默 token、群聊旁听、cron/heartbeat、MEMORY/USER/Honcho）。**分条发送、拟人打字延迟、情绪/好感度状态、B站字幕/弹幕读取与评论**则需要运行时代码，可用插件 hook、`ctx.llm`、skill 脚本或 MCP 工具来写。

### Cited Findings

**可用于活人感的 Hermes 插件 hook（全部来自官方 hooks 文档的 "Shipped plugin-hook catalog"）** — [features/hooks.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)
- `pre_llm_call`（Directive）："Once per turn before the loop; all valid string/{"context": ...} returns are joined and **injected into the user message**"，可拿到 `user_message`、`conversation_history`、`is_first_turn`、`platform`、`sender_id`。
- `transform_llm_output`（Transform）："Before post_llm_call and final delivery; **first non-empty string replaces the response**"。
- `post_llm_call`（Observer）：一轮成功结束后触发，可拿到完整的 `user_message`、`assistant_response`、`conversation_history`。
- `transform_tool_result`（Transform）："first string replaces the result"，在写入对话历史之前生效。
- `pre_gateway_dispatch`（Directive）：每条入站消息在入口处可返回 `skip` / `rewrite` / `allow`。`post_gateway_admission`：返回 `handled` 即**跳过本轮 agent**。
- `on_interim_message`、`on_stream_delta` 等流式观察 hook；另有 "Cache-safe system prompt sections"，插件段落固定在 volatile 尾部的 "after_memory" 锚点（[system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)）。
- 网关事件 hook（`~/.hermes/hooks/<name>/HOOK.yaml + handler.py`）：`agent:start`、`agent:end`、`session:compress`、`session:reset` 等。
- 插件内部可以发起独立 LLM 调用：`ctx.llm.complete(messages=[...], model=None, temperature=…)` / `complete_structured()` / `acomplete()`。文档写明适用于 "A multi-turn prompt (system + few-shot examples + user)" — [developer-guide/plugin-llm-access.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/plugin-llm-access.md)

**机制 → Hermes 挂点映射表**

| 活人感机制 | 性质 | Hermes 原生？ | 具体挂点 / 依据 |
|---|---|---|---|
| 人设 / 系统提示 | 纯 prompt | 原生 | `~/.hermes/SOUL.md`（slot #1，替换默认身份）；多人设用 profiles — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md) |
| 风格规则 / 反 AI 腔 | 纯 prompt | 原生 | SOUL.md 的 "What to avoid" 段（官方模板就有 Sycophancy、Hype language 等）；默认身份本身也禁止 "Great question" 类填充语 — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)；[prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py) |
| Few-shot 风格样例 | 纯 prompt | 原生（放置位置自选） | 写进 SOUL.md（每会话固定，缓存友好）；或放进 skill 的 references；或用 `pre_llm_call` 每轮注入少量样例 — [hooks.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)；[skills.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md) |
| 分条发送（多条短消息） | 运行时代码 | **未发现原生功能**：对 `human_delay`、`split_messages` 等关键词的代码搜索无结果；文档只提到平台长度/限流导致的 "chunks" | 需要插件。`transform_llm_output` 只能返回单个字符串，真正多条发送要在平台适配器层或用自定义发送工具实现（推断） — [messaging/index.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/index.md) |
| 打字延迟 | 运行时代码 | 部分原生：有 typing indicator（`typing_indicator` 默认 true）和流式编辑（Streaming = progressive message updates via editing），但没有按字数模拟的延迟 | 延迟逻辑需插件；拟人场景建议关闭流式编辑（推断） — [messaging/index.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/index.md) |
| 何时说话 / 静默 | prompt + 运行时 | 原生（有限制） | 静默 token：`[SILENT]`、`SILENT`、`NO_REPLY`、`NO REPLY`、`[静默]`/`静默`、`[沉默]`/`沉默`，整条回复须恰好是该 token。**限制**：对「人发来的消息」返回纯静默 token 时，网关会替换成简短提示，"because a message that needed a reply must not vanish"（Slack 未 @ 场景除外）— [messaging/index.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/index.md)。Telegram 的 `require_mention: true` + `observe_unmentioned_group_messages: true` 可以「看全群但只在被 @/回复时说话」，文档称之为 "OpenClaw/Yuanbao-style group behavior" — [messaging/telegram.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/telegram.md)。主动插话需用 `pre_gateway_dispatch`/`post_gateway_admission` 自写门控 |
| 主动消息 | prompt + 调度 | 原生 | cron（隔离会话、保留 SOUL 人设、投递到平台）— [cron.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/cron.md)；`/heartbeat`（同会话、空闲触发、可静默）— [heartbeat.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/heartbeat.md)；Bot Mode Routines / Bot 主动私信 — [bot-mode.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/bot-mode.md)。随机化发帖时间需额外代码（推断） |
| 长期记忆 / 用户画像 | 原生 + prompt | 原生 | MEMORY.md / USER.md（2,200 / 1,375 字符）+ `session_search` + Honcho 等外部 provider — [memory.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/memory.md)；[memory-providers.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/memory-providers.md) |
| 情绪 / 好感度状态 | 运行时代码 | **非原生** | `post_llm_call` 里用 `ctx.llm.complete_structured()` 更新状态 JSON，再由 `pre_llm_call` 注入 user 侧（不改 system，缓存友好）— [hooks.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)；[plugin-llm-access.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/plugin-llm-access.md) |
| 日记 / 自省 | prompt + 调度 | 半原生 | cron 每日任务 + skill（写文件或写 memory）；仓库里有 `agent/background_review.py` 和 curator 功能（[features/curator.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/curator.md)），但其是否适合日记用途未核实 |
| 读媒体（字幕/弹幕） | 工具 / 运行时 | 部分原生 | 内置 skill `skills/media/youtube-content`（YouTube 字幕 → 摘要）— [SKILL.md](https://github.com/NousResearch/hermes-agent/blob/main/skills/media/youtube-content/SKILL.md)；社区插件 **by2kb**（Bilibili/YouTube → 下载 + ASR → 转写/摘要，通过 `pre_gateway_dispatch` 拦截含链接的消息，依赖外部 CLI）— [plugin-catalog/by2kb.yaml](https://github.com/NousResearch/hermes-agent/blob/main/plugin-catalog/by2kb.yaml)。**未发现弹幕读取能力** |
| 写评论（B站） | 工具 / 运行时 | **非原生** | messaging 平台列表中没有 Bilibili 适配器（[messaging 目录](https://github.com/NousResearch/hermes-agent/tree/main/website/docs/user-guide/messaging)）；需自写 MCP/skill 脚本或用 browser / computer-use 功能（[features 目录](https://github.com/NousResearch/hermes-agent/tree/main/website/docs/user-guide/features)） |

**其他 harness 的对应挂点（简述）**
- **SillyTavern**：Author's Note 可按「深度」注入。depth 0 表示放在聊天历史最末尾，数字越大越往前；越靠近 prompt 底部，对下一条回复影响越大。社区常用 depth≈4 来强化角色特征（属于经验之谈，非测试结论）。有帖子称 Author's Note 可能破坏 Claude 的 prompt cache — [SillyTavern Docs: Author's Note](https://docs.sillytavern.app/usage/core-concepts/authors-note/)；[社区快照（Reddit r/SillyTavernAI 存档）](https://reddit.sentinel-team.org/posts/1s7ri3o/snapshots/2026-04-04T00%3A07%3A23.107607Z)
- **OpenClaw / Claude Code / Codex / OpenCode**：DeepSeek 同时提供 OpenAI 格式（`https://api.deepseek.com`）和 Anthropic 格式（`https://api.deepseek.com/anthropic`）的 Base URL，并为这些 harness 写了官方集成页 — [DeepSeek: Models & Pricing](https://api-docs.deepseek.com/quick_start/pricing)。Soul Spec 生态把 IDENTITY/STYLE 这类分文件人设统一合并成 SOUL.md 以兼容 Hermes — [ClawSouls](https://docs.clawsouls.ai/docs/guides/hermes)
- **MaiBot（麦麦，国产群聊拟人 bot）**：其 Maisaka 推理引擎里有 planner（动作循环：收集消息 → planner → 相似度检测 → 执行工具 → 可能继续循环）和 replyer（有「构建请求前」「发送给模型前」「回复生成后可改写或要求重新生成」三个钩子）；官网首页还宣传「智能发言时机」（读气氛决定说话或沉默）和学习群友说话风格 — [docs.mai-mai.org architecture（搜索摘要，原页已迁移）](https://docs.mai-mai.org/en/develop/architecture/maisaka-reasoning)；[docs.maibot.chat](https://docs.maibot.chat/)

### Inferences
- **可移植性分层建议**：
  1. 「人设包」（persona.md + style_rules.md + 20–50 条风格样例）保持 harness 无关的纯文本；Hermes 下合并进 SOUL.md，SillyTavern 下拆成角色卡与 Author's Note，Claude Code/OpenClaw 下放进各自的人设文件。
  2. 「行为层」（说不说、何时说、主动性）在 Hermes 中优先用原生的静默 token、群聊旁听、cron、heartbeat；做不到的再用 `pre_gateway_dispatch` 写门控。
  3. 「投递层」（分条、延迟、去 Markdown）和「状态层」（情绪/好感度/日记）写成一个小型 Hermes 插件，接口尽量通用（输入文本 → 输出若干条带延迟的消息），以便移植到其他 harness。
- 由于对人发来的消息返回纯静默 token 会被替换成提示，「被 @ 后选择不理人」在 Hermes 里做不到真正的已读不回。需要在 `post_gateway_admission` 返回 `handled` 来跳过整轮，或接受简短回应。
- B站场景（看视频 → 读字幕/弹幕 → 发评论）在 Hermes 中没有现成通道。by2kb 解决了「读字幕」，但弹幕和评论都要自写工具。评论还牵涉平台风控和账号规则，需要人工审视（推断，未查 B站规则）。

### Gaps
- 没有找到 Hermes 原生的「多条消息 + 拟人延迟」功能，也没确认是否存在可供插件调用的 `send_message` 工具或 adapter API 来实现多条发送；需要读 `gateway/` 源码确认。
- 没有核实 QQ 官方 Bot API 能否在群里收到未 @ 的消息（Hermes 文档只说需要 @）；这决定了 QQ 群「主动插话」能否实现。
- 没有实测 by2kb 插件的质量和许可。没有找到 B站弹幕/评论相关的 Hermes skill 或 MCP。

---

## 问题 3：DeepSeek 模型因素——2026 年可用版本、chat/reasoner、温度、思考模式、缓存，以及换 harness/换模型后如何迁移

### Takeaway
截至 2026-10-30，DeepSeek 官方 API 只有 **`deepseek-flash`（DeepSeek-V4.1-Flash-1017）** 和 **`deepseek-v4-pro`（DeepSeek-V4-Pro-0813）** 两个模型名。`deepseek-chat` / `deepseek-reasoner` 已在 2026-07-24 退役。两个模型都**默认开启思考模式**，而**思考模式下 temperature、presence_penalty、frequency_penalty 被静默忽略**。所以官方「闲聊 1.3 / 创作 1.5」的温度建议只有在**显式关闭思考**后才生效。Hermes 中可用 `/reasoning none`（或 effort=none）让 DeepSeek provider 发送 `thinking: disabled`。保留思考的话，可以用社区整理的「角色沉浸 / 纯分析」思维链指令来控制角色感。

### Cited Findings

**模型与版本时间线**
- 当前模型表：`deepseek-flash` → DeepSeek-V4.1-Flash-1017；`deepseek-v4-pro` → DeepSeek-V4-Pro-0813。上下文均为 1M，最大输出 384K，"Supports both non-thinking and thinking (default) modes"；均支持 JSON Output、Tool Calls、Responses API、Anthropic API。**Vision：flash 支持，v4-pro 不支持** — [DeepSeek: Models & Pricing](https://api-docs.deepseek.com/quick_start/pricing)
- 旧名 `deepseek-v4-flash` 和 `deepseek-v4-flash-vision-exp` 仍被接受，但对应模型已退役，请求由 V4.1-Flash-1017 提供并按 Flash 价计费 — [同上](https://api-docs.deepseek.com/quick_start/pricing)
- 价格（每 1M tokens，非高峰/高峰）：flash 输入缓存命中 $0.003/$0.006、未命中 $0.15/$0.3、输出 $0.6/$1.2；v4-pro 命中 $0.022/$0.044、未命中 $0.66/$1.32、输出 $1.98/$3.96。高峰时段为 UTC 周一至周五 01:00–04:00 和 06:00–10:00，非高峰价为高峰的一半。并发上限 flash 2500、pro 500 — [同上](https://api-docs.deepseek.com/quick_start/pricing)
- 新闻时间线：V4 Preview（2026-04-24，V4-Pro 1.6T 总参/49B 激活，V4-Flash 284B/13B，1M 上下文成为默认）→ V4-Pro GA（2026-08-13）→ V4-Flash-Vision-Exp（2026-08-21）→ V4.1-Flash（2026-09-10）→ V4.1-Flash-1017（2026-10-17）— [DeepSeek News: V4 Preview](https://api-docs.deepseek.com/news/news260424)；[News: V4.1-Flash-1017](https://api-docs.deepseek.com/news/news261017)
- **`deepseek-chat` 和 `deepseek-reasoner` "will be fully retired and inaccessible after Jul 24th, 2026, 15:59 (UTC)"**，退役前分别路由到 v4-flash 的非思考/思考模式 — [V4 Preview](https://api-docs.deepseek.com/news/news260424)。历史上 V3.1（2025-08-21）起 `deepseek-chat` 对应非思考、`deepseek-reasoner` 对应思考 — [V3.1 Release](https://api-docs.deepseek.com/news/news250821)
- V4.1-Flash-1017 官方宣称相比上一版"shorter outputs"，并针对 DeepSeek Harness、CodeBuddy、WorkBuddy 优化（建议 DeepSeek Harness ≥ v0.2.1-rc.1） — [News 261017](https://api-docs.deepseek.com/news/news261017)
- Hermes 源码会把退役的 `deepseek-chat`/`deepseek-reasoner` ID 在 `hermes_cli.model_normalize` 中重映射；DeepSeek provider 的 fallback 模型为 `("deepseek-v4-pro","deepseek-flash")`，辅助模型默认 `deepseek-flash` — [plugins/model-providers/deepseek/__init__.py](https://github.com/NousResearch/hermes-agent/blob/main/plugins/model-providers/deepseek/__init__.py)

**温度与采样**
- 官方温度页：默认 1.0。Coding/Math 0.0；Data Cleaning/Analysis 1.0；**General Conversation 1.3**；Translation 1.3；**Creative Writing/Poetry 1.5** — [DeepSeek: The Temperature Parameter](https://api-docs.deepseek.com/quick_start/parameter_settings)。注意：抓取到的这个页面侧边栏新闻只到 V3.2（2025-12），说明它可能是 V4 之前的旧页面，未随 V4 更新。另有旧快照（经 apidog 镜像）列出的是「对话 1.0 / 创作 1.25」，两个版本**互相冲突** — [apidog 镜像](https://apidog.com/apidoc/docs-site/747793/the-temperature-parameter-835828m0)
- **思考模式不支持 temperature、presence_penalty、frequency_penalty**，"setting these parameters will not trigger an error but will also have no effect"。top_p 行为：`deepseek-flash` 只在思考模式生效（0.95–1.0），非思考模式固定为 1.0；`deepseek-v4-pro` 在思考模式忽略用户值，非思考模式生效 — [DeepSeek: Thinking Mode](https://api-docs.deepseek.com/guides/thinking_mode)
- 开关方式：OpenAI 格式用 `extra_body={"thinking":{"type":"enabled|disabled"}}` + `reasoning_effort: low/high/max`；Responses 格式用 `reasoning.effort: none/low/high/max`（none 即关闭）。默认开启，effort 默认 high。映射关系：minimal→low、medium→high、xhigh→high、ultra→max — [Thinking Mode](https://api-docs.deepseek.com/guides/thinking_mode)
- `reasoning_content` 的回传规则：请求**带 tools** 时，之前各轮的 reasoning_content 应回传，并会拼进上下文；**不带 tools** 时无需回传，回传也会被忽略 — [Thinking Mode](https://api-docs.deepseek.com/guides/thinking_mode)
- V3 时代的「API 温度映射」：网页/App 的模型温度为 0.3，API 实现了映射，`T_model = T_api × 0.3（0 ≤ T_api ≤ 1）`，"if you call V3 via API, temperature 1.0 equals to the model temperature 0.3" — [HF: DeepSeek-V3-0324 README](https://huggingface.co/deepseek-ai/DeepSeek-V3-0324)（1<T_api≤2 区间的第二段公式在本次抓取中被截断，我记忆中是 `T_api − 0.7`，需复核）
- V4-Pro 模型卡建议**本地部署**采样参数为 `temperature = 1.0, top_p = 1.0`，Think Max 模式建议上下文至少 384K — [HF: DeepSeek-V4-Pro README](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)

**Hermes 中的 DeepSeek 推理/温度控制**
- Hermes 的 DeepSeek provider 对 V4+（以及无版本号的 `deepseek-flash`）**总是显式发送 `thinking` 开关**，用来避开 "reasoning_content echo trap"（不回传会在第一次工具调用后 HTTP 400）。effort 为 `none/false/disabled` 时发送 `{"thinking": {"type": "disabled"}}` — [deepseek/__init__.py](https://github.com/NousResearch/hermes-agent/blob/main/plugins/model-providers/deepseek/__init__.py)
- 全局 `agent.reasoning_effort` / `/reasoning <level>` 控制主模型思考，`auxiliary.<task>.reasoning_effort` 控制压缩等辅助任务 — [user-guide/configuration.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)
- 主模型温度：provider profile 有 `fixed_temperature` 字段（`None` 表示用调用方的值，`OMIT_TEMPERATURE` 表示不发送），并支持 "User overrides — replace a built-in without editing the repo" — [developer-guide/model-provider-plugin.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/model-provider-plugin.md)。插件内的 `ctx.llm.complete(..., temperature=…)` 可以自由设温度 — [plugin-llm-access.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/plugin-llm-access.md)
- DeepSeek 官方 Hermes 集成页推荐的模型是 `deepseek-v4-pro` — [DeepSeek: Hermes 集成](https://api-docs.deepseek.com/quick_start/agent_integrations/hermes)

**思考模式与角色感（社区）**
- GitHub 仓库 victorchen96/deepseek_v4_rolepaly_instruct（作者资料为 Victor Chen / Peking University；2026-04-24 创建，约 2,331 stars）给出 V4（flash/pro API 和 App 专家模式）的两种思维链控制指令。「**角色沉浸**」：思考中以第一人称、用括号写内心独白。「**纯分析**」：禁止内心独白，只做剧情分析和回复规划。FAQ 称指令"建议放在第一轮 user 消息末尾，这是训练时的注入位置，效果最稳定"，"指令只影响思考过程。但思考方式会间接影响回复——角色沉浸模式下情感更真实，纯分析模式下结构更稳定"；触发是概率性的 — [GitHub: deepseek_v4_rolepaly_instruct](https://github.com/victorchen96/deepseek_v4_rolepaly_instruct)
- B站有长文讨论 V4 发布后角色扮演变差，作者归因于注意力架构变化，并建议用分析型思维链加结构化预设来维持文风。**原文被验证码拦截，以上仅为搜索摘要** — [Bilibili cv48965247](https://www.bilibili.com/read/cv48965247)

**缓存**
- DeepSeek 磁盘上下文缓存默认开启。V4 使用滑动窗口注意力后，缓存以「前缀单元」为单位，**必须完整匹配**一个已持久化的前缀单元才算命中。持久化发生在：每个请求的用户输入末尾和模型输出末尾、系统检测到的跨请求公共前缀、长输入/输出的固定 token 间隔 — [DeepSeek: Context Caching](https://api-docs.deepseek.com/guides/kv_cache)
- 命中价约为未命中价的 2%（flash 非高峰 $0.003 对 $0.15） — [Pricing](https://api-docs.deepseek.com/quick_start/pricing)
- Hermes 的系统提示词按 stable → context → volatile 排列，正是为了让前缀缓存保持热；heartbeat 以 user 消息注入，"No system-prompt mutation" — [system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)；[heartbeat.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/heartbeat.md)

**换模型时 Hermes 自身的差异化注入**
- Hermes 对 DeepSeek/Qwen/GLM/GPT/Gemini 等注入工具强制和执行纪律块，对 Claude 不注入（"Claude does not exhibit these modes"）；gpt-5/codex 的系统提示以 developer 角色发送 — [prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
- Hermes 4 是混合推理模型，可通过 chat template 的 `thinking=True` 或特定 system prompt 打开推理 — [HF: Hermes-4-70B 卡（镜像）](https://huggingface.co/N0rv3ll/Hermes-4-70B)

### Inferences
- **「鱼灾」类 bot 的推荐参数（推断，待实测）**：对外说话的回复调用用 `deepseek-flash` 或 `deepseek-v4-pro`，**关闭思考**并设 temperature≈1.3（闲聊）到 1.5（段子、创作型评论），这样温度才真正生效。需要搜索、看视频、调工具的规划调用则开思考（effort high）。如果坚持只用一次调用且开思考，温度无效，只能靠「角色沉浸」类思维链指令（放在首轮 user 末尾）和 few-shot 来塑造语气。
- 许多旧教程写「deepseek-chat + temperature 1.3」，2026-07-24 之后直接照搬会失效（模型名已退役），或者看似生效其实无效（新模型默认开思考、温度被忽略）。这是复现失败的一个高概率隐性原因。
- **跨模型迁移温度不要照抄数字**：DeepSeek API 的 temperature 至少在 V3 时代存在 API→模型的映射（1.0→0.3），V4 本地建议 1.0。换成 Claude、Hermes-4 或其他 OpenAI 兼容端点时，1.3 可能明显过热。应以「风格样例上的盲评」重新标定，而不是复制数值。V4 API 是否仍做映射未查到官方说明。
- 换成 Claude 做主干时，Hermes 不会注入工具强制块，助手腔压力反而更小；但 Claude 自有的安全/助手人格倾向（见问题 4 的 Assistant Axis）可能让「网民口吻」更难维持。换成 Hermes-4 时，其宣传的 steerability 和低拒答理论上利于人设，但中文网络口语能力没有找到评测证据。
- 缓存设计建议：人设、风格规则、few-shot 放在稳定前缀（SOUL.md）；情绪状态、当前视频信息、近期记忆这类动态内容放在**末尾 user 消息**（Hermes 的 `pre_llm_call` 正是注入 user 消息）。这样既不破坏 DeepSeek 的前缀单元命中，又利用了「越靠后影响越大」。

### Gaps
- 官方温度页是否针对 V4 更新过、V4 API 是否仍做温度映射，都没有官方说明；新旧温度表互相冲突。
- 没找到 2026 年实践者对「V4-Pro 与 V4.1-Flash 谁的中文网民口吻更像人」的系统对比，也没找到量化测评。
- 没找到 Hermes `config.yaml` 中直接设置主模型 temperature 的文档化键名（只确认了 provider profile 的 `fixed_temperature` 和插件 `ctx.llm` 的 temperature 参数）。
- 没有打开 DeepSeek Harness 文档（https://deepseek-harness.github.io/deepseek-harness/en/guide/quickstart）核实其默认采样参数和人设机制，这部分由负责源生态的同事覆盖。

---

## 问题 4：「planner/actor + 人设 replyer」拆分架构是否稳健？长会话的人设漂移、助手腔注入、工具输出污染风格、压缩毁人设等失败模式及缓解

### Takeaway
学术与工程证据都表明，**人设会随对话轮数和话题类型漂移**：8 轮内即可显著漂移，原因与注意力衰减有关；元反思类和情绪化对话会把模型推离或拉回默认的 Assistant 人格。国产拟人群聊 bot（MaiBot）已采用 planner 与 replyer 分离的结构。**但我没有找到直接对照实验证明「拆分 replyer」能稳定风格**，这一点属于工程推断。Hermes 提供了实现拆分所需的全部挂点：`transform_llm_output` + `ctx.llm`、`pre_llm_call`、`transform_tool_result`，以及关闭工具强制、关闭中途旁白。

### Cited Findings

**人设漂移的证据**
- Li 等（2024，arXiv 2402.10962）用两个人设 chatbot 自对话的基准测量漂移，在 LLaMA2-chat-70B 等模型上发现 "significant persona drift within eight rounds"，并归因于 transformer 的注意力衰减。他们提出 split-softmax（推理时放大对 system prompt 的注意力，无需训练），效果优于两个基线 — [arXiv 2402.10962](https://arxiv.org/abs/2402.10962)；[Harvard VCG 页面](https://vcg.seas.harvard.edu/publications/measuring-and-controlling-persona-drift-in-language-model-dialogs)
- Anthropic 等（2026-01，arXiv 2601.10387，"The Assistant Axis"）发现模型的人格空间存在主导方向 "Assistant Axis"，用来衡量偏离默认助手模式的程度。要求元反思的对话和情绪脆弱用户的对话会驱动漂移；他们提出沿该轴做 activation capping 来稳定行为 — [arXiv 2601.10387](https://arxiv.org/html/2601.10387)；[The Decoder 报道](https://the-decoder.com/anthropic-study-finds-that-role-prompts-can-push-ai-chatbots-out-of-their-trained-helper-identity/)（部分细节来自二手报道）
- 有 NeurIPS 2025 海报用多轮 RL 训练人设一致性 — [NeurIPS 2025: Consistently Simulating Human Personas with Multi-Turn RL](https://neurips.cc/virtual/2025/poster/119491)（仅见标题和摘要）

**拆分架构的工程先例**
- MaiBot（Maisaka 引擎）在内部思考循环中先跑 planner（动作循环 + 工具），回复由 replyer 层生成，并提供「生成后改写或要求重新生成」钩子；官网宣传「智能发言时机」和风格学习 — [docs.mai-mai.org（搜索摘要）](https://docs.mai-mai.org/en/develop/architecture/maisaka-reasoning)；[docs.maibot.chat](https://docs.maibot.chat/)
- SillyTavern 用深度注入（Author's Note）把人设提醒放在靠近末尾的位置，因为越靠近底部影响越大 — [SillyTavern Docs](https://docs.sillytavern.app/usage/core-concepts/authors-note/)
- DeepSeek V4 的「纯分析」思维链指令（像导演一样冷静规划：场景、策略、字数）与「角色沉浸」（像演员一样入戏）形成对照。纯分析让「结构更稳定」，角色沉浸让「情感更真实」，相当于在一次调用内部做了 planner/actor 区分 — [deepseek_v4_rolepaly_instruct](https://github.com/victorchen96/deepseek_v4_rolepaly_instruct)

**Hermes 中的具体失败模式（带源码/文档依据）**
1. **harness 注入助手腔**：DeepSeek 默认被注入 TOOL_USE_ENFORCEMENT（「每条回复要么调工具要么交付最终结果」）和 EXECUTION_GUIDANCE — [prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)；[system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
2. **中途旁白泄漏**：`interim_assistant_messages` 在 Telegram 默认开启，会把「我先去看看…」之类的过程叙述作为消息发出；`long_running_notifications` 会发「⏳ Working — N min」 — [messaging/index.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/index.md)
3. **工具输出污染风格**：工具结果（网页抓取、字幕全文、终端输出）进入对话历史并在后续轮次重复发送。Hermes 有 `proactive_prune_tokens` 对旧的大工具结果做去重/摘要（默认关），`transform_tool_result` 可在入历史前改写 — [configuration.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)；[hooks.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)
4. **上下文压缩**：默认 `compression.threshold: 0.50`（达到上下文 50% 时压缩）、`target_ratio: 0.20`、`protect_last_n: 20`。压缩摘要是单独的 LLM 调用（`auxiliary.compression`，可换成别的模型），并有 reply-anchor 保证刚发出的那条 assistant 回复留在尾部。SOUL.md 位于系统提示中，不参与摘要；压缩会触发系统提示重建 — [configuration.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)；[agent/conversation_compression_reply_anchor.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/conversation_compression_reply_anchor.py)；[system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/system_prompt.py)
5. **子代理丢人设**：delegation/subagent 上下文设置了 `skip_context_files` 时回退到默认身份 — [personality.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/personality.md)
6. **记忆写成指令导致后续人设被覆写**：Hermes 记忆指导明确警告命令式条目会在后续会话被当成指令重读 — [prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
7. **长期会话不切换**：记忆只在会话边界生效，网关场景要用户自己 `/new`。`idle_compact_after_seconds` 可在长时间空闲后先压缩再回复 — [memory.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/memory.md)；[configuration.md](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)

### Inferences

**推荐的 harness 无关拆分架构（基于以上证据的工程推断，未经对照实验验证）**
```
入站事件(弹幕/评论/群消息/定时tick)
  → [Gate] 规则+小模型判定: skip / observe / respond / proactive      (Hermes: pre_gateway_dispatch / post_gateway_admission / cron / heartbeat)
  → [Planner] 带工具、开思考(effort high)、助手身份即可:
        看视频字幕、查资料、读记忆 → 输出结构化 JSON {should_speak, intent, facts[], target, mood_delta}
        (Hermes: 主 agent 循环; persona profile 中关闭 interim/tool_progress)
  → [Replyer] 无工具、关思考、T≈1.3–1.5(DeepSeek API 标度):
        输入 = 人设包(SOUL) + 20–50 条风格样例 + 最近 N 条「纯聊天」历史(不含工具输出) + planner 的 intent/facts + 当前情绪/好感度
        (Hermes: transform_llm_output 内用 ctx.llm.complete(...) 重写最终回复)
  → [Post] 去 Markdown/去 AI 口头禅正则、长度裁剪、分条、按字数延迟、静默 token
        (Hermes: 自写平台/投递插件)
  → [Reflect] 更新情绪/好感度/日记、写 USER.md 陈述式事实      (Hermes: post_llm_call + cron 日记)
```
- **为什么拆分可能更稳**：
  1. replyer 的上下文里没有工具输出和工具强制指令，「助手腔」的两大来源被物理隔离。
  2. replyer 每次都重新拼接人设和样例，相当于「每轮人设再注入」，对抗 Li 等发现的注意力衰减。
  3. planner 可以用适合推理的设置（开思考），replyer 用适合口语的设置（关思考、高温），绕开「思考模式下温度无效」的 DeepSeek 约束。
  4. 换主干模型时只需重标定 replyer 的温度和样例，planner 几乎不受影响。
- **单次调用替代方案**（成本更低但稳定性更弱）：persona profile 关闭 `tool_use_enforcement`、`execution_guidance` 和中途旁白；`pre_llm_call` 每轮在 user 消息末尾注入 2–3 行人设提醒加当前状态（相当于 SillyTavern depth 0–4 的 Author's Note，且不破坏 system 前缀缓存）；`transform_tool_result` 把长工具输出压缩成要点；`transform_llm_output` 做正则清洗。
- **压缩的缓解**：1M 上下文模型可通过 `compression.threshold` 和 `threshold_tokens` 控制压缩时机；把 `auxiliary.compression` 指向便宜模型并低 effort；人设关键信息只放在 SOUL.md（不会被摘要），不要依赖历史里的早期 assistant 回复充当风格锚。用「固定风格样例」取代「历史回复」作为风格锚，压缩就毁不掉风格。
- **漂移监控**：可以借鉴 Li 等的思路，定期用固定探针问题测量人设一致性，或用 `post_llm_call` 记录回复做离线风格打分。

### Gaps
- **没有找到**公开的对照实验（论文或社区 A/B）直接证明「planner + 独立 persona replyer」比单次调用更能稳定 LLM agent 的风格；MaiBot 只是工程先例，未见量化数据。
- 没有找到 Hermes 社区（Reddit、linux.do、知乎）关于用 Hermes 跑长期拟人 bot 并遇到人设漂移的一手帖子。中文搜索只命中 CSDN 上关于 Hermes「问一句答一句、跟 ChatGPT 差不多」的泛泛讨论（[CSDN](https://deepseek.csdn.net/6a32069510ee7a33f27e4a09.html)，只看了搜索摘要）。
- Hermes 的压缩摘要提示词是否会把 assistant 的口语风格改写成第三人称/助手口吻，没有读到 `context_compressor_summary.py` 中摘要 prompt 的全文。
- Assistant Axis 和 persona drift 论文的实验对象不是 DeepSeek V4；V4 的漂移速率没有找到数据。
