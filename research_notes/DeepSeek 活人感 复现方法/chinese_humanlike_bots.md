# 中文开源"拟人/活人感"聊天机器人与角色扮演社区的具体技术（AstrBot 与 DeepSeek-harness 生态之外）

> 调研日期 2026-10-30。主要依据是用 `gh api` 只读拉取的 GitHub 源码（MaiBot、KouriChat、WeChatBot_WXAUTO_SE、Neo-MoFox、N.E.K.O 等），辅以少量网页。所有引用的 prompt 保留原文。版本与日期都按抓取时的仓库状态记录，这些项目更新很快。
> 仓库概况（抓取时）：Mai-with-u/MaiBot ★6723，GPL-3.0，最新 changelog 为 1.3.18（2026-10-28）；KouriChat/KouriChat ★3299，最新 release v1.5.0.1（2026-09-06）；Project-N-E-K-O/N.E.K.O ★3141，Apache-2.0；MoFox-Studio/Neo-MoFox ★121，AGPL-3.0；iwyxdxl/WeChatBot_WXAUTO_SE ★880，已"停止维护"。

## 问题一：这些项目认为"像人"必不可少的架构组件有哪些？

### Takeaway
国内最成熟的拟人项目 MaiBot（麦麦）有一个核心判断：活人感主要来自聊天文本之外的"行为层"，模型本身的作用排在后面。它的拟人栈包括：消息防抖与合并、要不要说话的门控（概率或学习模型加 planner）、planner 与 replyer 分离、从群友那里学表达方式和黑话、规则或 LLM 分段、带纠正的错别字、模拟打字延迟、偶尔引用回复、表情包、长期记忆和人物画像，主动/空闲退避则是可选项。微信伴侣类项目（KouriChat 系）更依赖以下手段：人设卡、反斜杠分段、随机发送间隔、消息队列合并、定时主动消息和安静时段。

### Cited Findings

**MaiBot / 麦麦（Mai-with-u/MaiBot，原 MaiM-with-u；当前称 "MaiSaka"）**
- 设计哲学：README 写明项目目标"不是为了写一个功能齐全的机器人，而是一个尽可能让人感知到真实的类人存在"，核心原则是"最像而不是好"。用户需要的是"一个会犯错的，拥有自己感知和想法的'生命形式'"。README 的功能列表第一条就是"没有人喜欢 GPT 的语言风格……不是长篇大论或者 markdown 格式的分点，而是或长或短的闲谈"，第二条是"懂得在合适的时间说话……在合适的时候开口，在合适的时候闭嘴"。— [MaiBot README](https://github.com/Mai-with-u/MaiBot/blob/main/README.md)
- 架构演进：1.0.0（2026-06-12）把核心改成 Agent 式（"麦麦现在和一个Agent一样工作，但是仍保有优秀的回复质量"），同时上线中期记忆（长对话压缩成摘要）和 A_Memorix 长期记忆（向量、图关系、BM25、PageRank 融合检索）以及"人物画像"。0.12.0（2025-12-21）"planner和replyer现在开启联动"，加入"思考力度机制，动态控制回复时间和长度"，还新增了"麦麦做梦功能"，并且**"移除频率自动调整"和"移除情绪功能"**。— [MaiBot changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- 文件结构上 heartflow（心流）这个名字还在：`src/chat/heart_flow/heartflow_manager.py`。决策链路则拆成 `src/maisaka/turn_trigger/{dynamic_gate,gates,idle_backoff,reply_likelihood,scheduler}.py`，另有 `src/chat/debounce/`、`src/chat/utils/typo_generator.py`、`src/chat/utils/llm_sentence_splitter.py`、`src/learners/{expression_learner,jargon_learner,jargon_miner}.py`、`src/maisaka/attention_drift.py`、`src/maisaka/quote/`、`src/emoji_system/` 等模块。— [MaiBot 仓库树](https://github.com/Mai-with-u/MaiBot/tree/main/src)
- **组件 1：消息防抖（模拟"等对方说完"）。** 群聊默认 `message_debounce_mode="natural"`，含义是"安静 2 秒、活跃 5 秒，同人续发 2 秒、换人 0.5 秒"。另外"裸图开启新批次时至少等 4 秒……整批从第一条实际收到起最多 8 秒，@或提及后最多 1 秒"。私聊有独立的一组档位。— [official_configs.py ChatReplyTimingConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 2：何时说话的门控，用统计模型拟合回复概率。** `reply_likelihood.py` 是一个 logistic 回归。注释说明"权重由无必要性门控期间的 Planner 首轮记录拟合得到"，样本只包括"有新的外部消息且没有 @ 麦麦"的轮次，被 @ 的消息直接强制触发。各特征权重：截距 -1.2005，提到 bot 名字 +1.5852，@ 了别人 -1.1157，带问号 +0.4807，纯表情或图片占位 -1.0281，最近 5 分钟 bot 自身发言占比 +2.6525，最近 5 分钟消息数每条 -0.0367，log(距 bot 上次发言秒数) +0.0571，本批新消息数每条 +0.1012。— [reply_likelihood.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/turn_trigger/reply_likelihood.py)
- **组件 3：空闲退避。** 群聊中 planner 连续空闲（决定不说话）后，消息触发会按 `base_seconds * 2**(count-start_count)` 指数推迟，上限为 cap。待处理消息数达到 bypass 阈值时可以跳过退避。— [idle_backoff.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/turn_trigger/idle_backoff.py)
- **组件 4：群聊频率旋钮。** `talk_value`（0~1，默认 1，"越小越安静"），还可以按时段或按群设置 `talk_value_rules`。0.11.6 模板示例：`{ time = "00:00-08:59", value = 0.8 }`、`{ time = "09:00-22:59", value = 1.0 }`、某群晚间 0.6、某私聊 0.3。— [0.11.6 bot_config_template.toml](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)；[official_configs.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 5：planner 与 replyer 分离。** Planner 用 `planner` 模型，带工具：`reply`、`wait`、`stop`、`query_memory`、`view_forward_message`、`tool_search`、子代理等。规定"发言必须通过reply工具"，并通过 `reply_reference` 把"回复方向指引"传给 Replyer。Replyer 用单独的模型，只负责"日常且口语化"地写出那句话。另有一个"回复补充器"与 Replyer 并行运行，负责安排引用、图片、表情和 @。— [maisaka_chat.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_chat.prompt)；[maisaka_chat_passive.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_chat_passive.prompt)；[maisaka_replyer.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_replyer.prompt)
- 人设拆成三个字段：`personality`（"麦麦的人格和身份设定"），`behavior_style`（"Planner 使用的行动准则，例如何时参与聊天、如何观察局面以及何时保持安静"），`reply_style`（"麦麦平时说话的风格"）。changelog 记载"分离行为风格和人设，更精准的控制planner和replyer的行为"。— [official_configs.py PersonalityConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)；[changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- **组件 6：表达学习（从群友学说话）。** LLM 从群聊记录里抽取"当 A 时，可以 B"格式的规则。生成时有两种模式：快速档"加权随机抽取表达，直接供回复参考，不额外调用选择模型"（权重按使用次数 count 线性映射到 1~5 倍）；精细档"使用向量召回，再由 LLM 按语境筛选"。写入前默认先自检，即 `expression_self_reflect=True`，"写入表达方式前先让 AI 检查，减少学到奇怪内容"。— [official_configs.py ExpressionConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)；[weighted_sampling.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/learners/weighted_sampling.py)
- **组件 7：黑话挖掘。** 先抽候选词（最多 30 个），再结合上下文推断含义并存库。运行时用"基于当前上下文的黑话机械匹配"，把命中的词条与含义作为一条 `<jargon>` 参考消息注入上下文。— [learn_jargon.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/learn_jargon.prompt)；[jargon_inference_with_context.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/jargon_inference_with_context.prompt)；[jargon_context_matcher.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/jargon_context_matcher.py)
- **组件 8：中文错别字生成器（会自己纠正）。** 基于 pypinyin 和 jieba 词频，在同音或近声调、字频相近的字之间替换（`math.exp(-3 * freq_diff / max_freq_diff)` 衰减）。后处理时如果一句产生了可纠正的错字：50% 概率发出错字版并紧跟一条纠正消息（可引用前一条），另外 50% 改发原句。配置说明写的是"让麦麦偶尔打错字，更像真人聊天"。— [typo_generator.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/utils/typo_generator.py)；[utils.py `_apply_response_post_processing`](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/utils/utils.py)；[official_configs.py ChineseTypoConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 9：分段发送。** 默认用规则断句（`mode="rule"`），可选用 LLM 断句。规则断句后还会做概率合并：文本少于 12 字时 `split_strength=0.2`（合并概率 0.8），12~32 字为 0.6，更长为 0.7。不到 3 个字的文本有 1% 概率被逐字拆开发。— [utils.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/utils/utils.py)
- **组件 10：打字延迟。** `calculate_typing_time(chinese_time=0.3, english_time=0.15)` 按字符累加（注意：docstring 写的是"默认为0.2秒"，与代码不一致）。只有一个汉字时返回 `chinese_time * 3 + 0.3  # 加上回车时间`，表情或图片固定 1 秒，再乘以 `typing_speed`（0 最快，1 默认，2 更慢）。— [utils.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/utils/utils.py)；[official_configs.py ResponsePostProcessConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 11：引用回复频率。** 群聊和私聊默认都是 `occasionally`。私聊会"结合用户连发条数、发言间隔和回复目标位置随机决定"，"经常引用的概率为偶尔引用的两倍"。— [official_configs.py ChatReplyStyleConfig](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 12：表情包。** 会收集群里的表情（`steal_emoji`），用 VLM 描述后入库。发送时把候选表情拼成 N×N 网格图让模型挑序号（见问题二）。0.11.6 时代的默认值是 `emoji_chance = 0.4`（"麦麦激活表情包动作的概率"）。— [emoji_selection.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/emoji_selection.prompt)；[0.11.6 bot_config_template.toml](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)
- **组件 13：人格随机多样性。** 0.11.6 有 `states` 列表（如"是一个赛博网友，最近很想吐槽人。"），配合 `state_probability = 0.3`（"每次构建人格时替换personality的概率"）。当前版本改为 `multiple_reply_style` 加 `multiple_probability`（默认 0），即"随机启用备用风格的概率"。— [0.11.6 模板](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)；[official_configs.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **组件 14（实验，默认关闭）：注意力漂移。** 让 bot "更容易被有趣的新话题、梗或反差点吸引"，分 subtle/active/scattered/wild 四档，并配合"回钩策略"和"短反应风格"。— [attention_drift.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/attention_drift.py)
- 关系与情绪的去向：0.11.6 模板里还有 `[relationship] enable_relationship = true` 和 `[mood] enable_mood = false`（`emotion_style = "情绪较为稳定，但遭遇特定事件的时候起伏较大"`）。0.12.0 移除了情绪功能，1.0.0 改用 A_Memorix 的"人物画像"，"支持人物事实写回证据、画像自动注入"。— [0.11.6 模板](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)；[changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)

**Neo-MoFox（MoFox-Studio/Neo-MoFox，MaiBot 系分支 MoFox-Core 的重构版）**
- MoFox-Core 的仓库描述是"多功能、拟人效果优化的MaiBot fork项目"（最近一次推送 2026-06-11），Neo-MoFox 的描述是"目标……是创造一个真正的数字生命体"（最近一次推送 2026-10-28）。— [MoFox-Core](https://github.com/MoFox-Studio/MoFox-Core)；[Neo-MoFox](https://github.com/MoFox-Studio/Neo-MoFox)
- 何时说话分三层：(1) 程序化概率门。算法是"基础概率 + 强/弱提及 + 未读加成 + next_tick bonus"，其中 next_tick bonus 指"发送成功后给下一轮的概率加成"；(2) 语义兴趣度，用 TF-IDF 加逻辑回归，自动训练时采样最近 7 天、最多 1000 条消息；(3) LLM sub-agent 输出 `should_respond` JSON。兴趣门默认值：回复阈值 0.72，非回复动作阈值 0.55，语义权重 0.6，提及权重 0.4，强提及（@、被回复、私聊）记 2.0 分，弱提及（文本命中名字）记 0.8 分，每次不回复阈值降低 0.02（最多累计 5 次），回复后阈值先降 0.2，再按 0.8 衰减，最多持续 5 次。— [probability_gate.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/utils/probability_gate.py)；[default_chatter config.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/components/config.py)；[仓库树 semantic_interest/](https://github.com/MoFox-Studio/Neo-MoFox/tree/HEAD/plugins/default_chatter/semantic_interest)
- 回复后有冷却：`stop_conversation` 工具可以指定一段冷却时间，"期间新消息不会触发回复"。— [default_chatter config.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/components/config.py)

**KouriChat（微信/OneBot 情感陪伴）及其分支 WeChatBot_WXAUTO_SE**
- KouriChat 1.5 已完全重构，README 的功能列表包括"沉浸式角色扮演（支持群聊）""智能对话分段 & 情感化表情包""持久记忆存储""OneBot协议兼容"。现在的仓库描述是"【禁止接入微信、QQ等腾讯系软件】接入第三方平台"。— [KouriChat README (main)](https://github.com/KouriChat/KouriChat)
- 1.5 的人设包 `.krp` 包含 `heart.prompt`（"心核人设文本"）、`reason.prompt`（"智核人设文本"）、`expressions/index.json`（表情包加"给 AI 的解释"）、`tools/`。人设文本因此分成"心"（情感表达）和"智"（推理）两个核，思路上与 MaiBot 的 replyer/planner 分离相同。— [kourichat/persona.py](https://github.com/KouriChat/KouriChat/blob/main/kourichat/persona.py)
- 旧版 WeChat-wxauto 分支的拟人机制：消息先进入缓存队列合并处理；合并时只把上一轮对话作为"只是提醒"的上下文注入；回复按 `{'\\', '|', '￤', '\n'}` 分割成多条，每条之间 `time.sleep(random.uniform(0.5, 1.5))`；另外有自动消息倒计时和安静时段。— [handlers/message.py (WeChat-wxauto)](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/handlers/message.py)；[config.json.template](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/config/config.json.template)
- WeChatBot_WXAUTO_SE（KouriChat 修改版）把拟人参数都放进了配置：消息队列等待 `QUEUE_WAITING_TIME = 7`（秒）；发送间隔按"间隔时间 = 字数 * (平均时间 + 随机时间)"计算，`AVERAGE_TYPING_SPEED = 0.2`、`RANDOM_TYPING_SPEED_MIN = 0.05`、`RANDOM_TYPING_SPEED_MAX = 0.1`；`SEPARATE_ROW_SYMBOLS = True`；表情包发送概率 `EMOJI_SENDING_PROBABILITY = 25`；主动消息倒计时 1~2 小时，安静时段 22:00–8:00；记忆用"综合评分公式：0.6*重要度 - 0.4*(存在时间小时数)"，`MAX_MEMORY_NUMBER = 50`；`REMOVE_PARENTHESES` 用来"强制移除括号当中的内容"。— [WeChatBot_WXAUTO_SE config.py](https://github.com/iwyxdxl/WeChatBot_WXAUTO_SE/blob/HEAD/config.py)

**N.E.K.O（Project-N-E-K-O/N.E.K.O，桌面猫娘伴侣，"reaching out first"）**
- 仓库描述强调主动搭话："A catgirl who lives with you in real time — reaching out first, sharing your media…"。主动触发源有 B站首页推荐、微博热议、视频、屏幕截图、当前窗口加搜索、关注的 UP 主动态、音乐等，每种都有单独的 prompt。生成主动消息后还要再过一遍"文本清洁"改写。— [N.E.K.O 仓库](https://github.com/Project-N-E-K-O/N.E.K.O)；[prompts_proactive.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_proactive.py)
- **反 slop 过滤器（prompt 侧改写自身历史中的套话）。** `prompts_slop.py` 的说明是：这些规则"rewrite recurring AI-writing cliches in the assistant turns of the *outgoing* prompt (promptOnly — the persisted history and what the user sees are untouched), so the model stops imitating its own stock phrases"。规则从第二次出现开始改写（`SLOP_REPEAT_THRESHOLD = 2`），中文有 30 条，受产品里的"Natural Speech"开关控制。— [prompts_slop.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_slop.py)

**其他（只确认了元数据，未深读源码）**
- morettt/my-neuro（★1411，Neuro-sama 风格的桌面 AI 伴侣，宣称"respond in just 1 second"）；KroMiose/nekro-agent（★1161，多人互动场景 Agent 框架）；KroMiose/nonebot_plugin_naturel_gpt（★515，NoneBot 人格自定义/切换插件，最近推送 2025-06-24）；FuQuan233/nonebot-plugin-llmchat（★52）。— [my-neuro](https://github.com/morettt/my-neuro)；[nekro-agent](https://github.com/KroMiose/nekro-agent)；[naturel_gpt](https://github.com/KroMiose/nonebot_plugin_naturel_gpt)
- gejifeng/Hermes_Soul_patch（★17，MIT）直接面向 Hermes，描述为"Hermes companion plugin: 兴趣驱动的连续生活、有来源的记忆、联系感知与按块外发、主人审批的小红书互动"。和用户的目标（在 Hermes 里复现活人感）直接相关，但我没有读它的源码。— [Hermes_Soul_patch](https://github.com/gejifeng/Hermes_Soul_patch)

### Inferences
- 这些项目的共识是：活人感至少一半来自不说话或晚点说话，也就是防抖、门控、退避、冷却。MaiBot 甚至用自己 planner 的日志拟合了一个"会不会回复"的逻辑回归，Neo-MoFox 则用"不回复就降阈值、回复后先降再衰减"的方式模拟话头延续。在 Hermes 一类 harness 里复现时，可以把"何时说话"做成独立于主模型的轻量门（规则加概率），不必每条消息都唤醒大模型。
- planner/replyer 分离（或 KouriChat 1.5 的"智核/心核"）是为了让推理型模型的"八股分析腔"不进入最终台词：planner 可以用思考模型，replyer 只拿到一段"回复方向指引"，再用口语风格单独写。
- 输出后处理（分段、错字加自我纠正、按字数计算的打字延迟、偶尔引用）完全不依赖模型和 harness，是成本最低、最通用的一层。
- MaiBot 在 0.12.0 删掉了情绪系统和频率自动调整，说明显式的"情绪数值或状态机"在实践中收益不明显。后来的做法是把情绪写进人设和行为风格文本，再靠记忆和人物画像体现关系。这一点属推断，changelog 没有写删除原因。

### Gaps
- MaiBot 旧版 `willing`（聊天意愿）模块的具体公式没有回溯核对。当前代码里这一功能已由 talk_value、reply_likelihood 和 dynamic_gate 取代，dynamic_gate.py 的阈值表没有读。
- MaiBot 文档站已迁到 docs.maibot.chat（旧站 docs.mai-mai.org 显示"麦麦文档已迁移"），新站的架构文档没有读。— [docs.mai-mai.org 迁移页](https://docs.mai-mai.org/en/develop/architecture/expression-learning)
- MoFox-Core（旧版）的亲和力流、主动思考、睡眠和日程等模块，以及 my-neuro、nekro-agent、Hermes_Soul_patch 的源码，因时间预算没有读。
- 通过 GitHub 仓库搜索没能找到"nonebot 拟人"类插件，中文分词搜索返回为空，所以 NoneBot 生态覆盖不足。

## 问题二：逐字的 prompt 模式（人设格式、说话风格、"不要……"负面约束、长度限制、示例）

### Takeaway
各项目的 prompt 高度趋同，可以归纳为"身份一行 + 说话风格一行 + 场景注意事项 + 大量负面约束 + 输出只要台词"。高频词组有：日常、口语化、平淡、简短、不要太有条理、参考贴吧/微博、不要没话找话、不要每条都回、读空气。这些项目还会用学到的"当 X 时，可以用 Y"句式，以及禁用套话表，来压制模型自带的腔调。

### Cited Findings

**MaiBot 当前默认人设字段（official_configs.py）** — [源码](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- `personality` 默认值：`"是女大学生，现在正在上网聊天。一般不包容且善良，但有时候有点一反常态，表现出锐利攻击性。善于用人类的角度思考问题，聊天比较日常。"`（原文就是"一般不包容且善良"，可能是笔误。）
- `behavior_style` 默认值，在 personality 后追加：`"……不会没话题硬找话题，具有足够的自主性，会自行搜集信息理解内容。"`
- `reply_style` 默认值：`"你的风格平淡简短，可以参考贴吧的回复风格。不滥用比喻或者生硬句子。视情况省略主语或者进行倒装，风格较为随意。"`
- `multiple_reply_style` 备用风格示例：`"你的风格平淡但不失讽刺，很简短,很白话。可以参考贴吧，微博的回复风格。"`、`"用1-2个字进行回复"`、`"用1-2个符号进行回复"`、`"带点翻译腔，但不要太长"`。
- `group_chat_prompt` 默认值（群聊通用注意事项）：
  ```
  下面是聊天中的的内容，聊天中包含文字，图片和表情包等消息。
  回复尽量简短一些。最好一次对一个话题进行回复，但必须考虑不同群友发言之间的交互，免得啰嗦或者回复内容太乱。
  根据聊天内容自由发挥，但是要日常不浮夸，不要刻意找话题。
  不用刻意回复其他人发送的表情包。你可以适当发送表情包表达情绪。如果有人不喜欢你或者不理你，就不要强行回复，回复前读空气。不要每个人的消息都回复，适当回复其他话题。可以适当回过头回复之前用户的消息。
  ```
- `private_chat_prompts` 默认值：`"你正在聊天……回复尽量简短一些。请注意把握聊天内容。\n请考虑对方的发言频率，想法，思考自己何时回复以及回复内容。"`

**MaiBot 0.11.6 旧版人设（社区里流传最广的一版）** — [0.11.6 bot_config_template.toml](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)
- `personality = "是一个女大学生，现在在读大二，会刷贴吧。"`（注释："建议120字以内，描述人格特质 和 身份特征"）
- `reply_style = "请回复的平淡一些，简短一些，说中文，不要刻意突出自身学科背景。可以参考贴吧，知乎和微博的回复风格。"`
- `interest = "对技术相关话题，游戏和动漫相关话题感兴趣，也对日常话题感兴趣，不喜欢太过沉重严肃的话题"`
- `plan_style`：`"3.请控制你的发言频率，不要太过频繁的发言\n4.如果有人对你感到厌烦，请减少回复\n5.如果有人对你进行攻击，或者情绪激动，请你以合适的方法应对"`
- 关键词反应（否认是 AI）：`{ keywords = ["人机", "bot", "机器", "入机", "robot", "机器人", "ai", "AI"], reaction = "有人说你是人机或者机器人，否定这一事实，攻击或反问他。注意，只有认为你是机器人才回复，否则不要否认" }`

**MaiBot 0.11.6 Planner（reply / no_reply 二选一）** — [planner.py@0.11.6-beta](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/src/chat/planner_actions/planner.py)
```
reply
动作描述：
1.你可以选择呼叫了你的名字，但是你没有做出回应的消息进行回复
2.你可以自然的顺着正在进行的聊天内容进行回复或自然的提出一个问题
3.不要回复你自己发送的消息
4.不要单独对表情包进行回复
...
no_reply
动作描述：
保持沉默，不回复直到有新消息
控制聊天频率，不要太过频繁的发言
...
先输出你的简短的选择思考理由，再输出你选择的action，理由不要分点，精简。
```

**MaiBot 0.11.6 Replyer（经典群聊回复 prompt）** — [replyer_prompt.py@0.11.6-beta](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/src/chat/replyer/prompt/replyer_prompt.py)
```
{knowledge_prompt}{tool_info_block}{extra_info_block}
{expression_habits_block}{memory_retrieval}{jargon_explanation}
你正在qq群里聊天，下面是群里正在聊的内容，其中包含聊天记录和聊天中的图片
其中标注 {bot_name}(你) 的发言是你自己的发言，请注意区分:
{time_block}
{dialogue_prompt}
{reply_target_block}。
{planner_reasoning}
{identity}
{chat_prompt}你正在群里聊天,现在请你读读之前的聊天记录，然后给出日常且口语化的回复，平淡一些，{mood_state}
尽量简短一些。{keywords_reaction_prompt}请注意把握聊天内容，不要回复的太有条理，可以有个性。
{reply_style}
请注意不要输出多余内容(包括前后缀，冒号和引号，括号，表情等)，只输出一句回复内容就好。
不要输出多余内容(包括前后缀，冒号和引号，括号，表情包，at或 @等 )。
现在，你说：
```
（这个模板的结构是先放各种参考信息，再放聊天记录，最后放身份、风格和格式约束，并以"现在，你说："收尾。）

**MaiBot 当前 Planner（Maisaka，工具调用式）** — [maisaka_chat.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_chat.prompt)
```
你的名字是{bot_name}{bot_alias_notice}。你的性格：{personality}。你的行为风格是：{behavior_style}{bot_nickname_notice}

在当前场景中，不同的人正在聊天互动。请你对当前场景和输出规则来进行分析。你需要对下一步行为做出决策，结合行为风格、当前情况和可用工具做出决策。
现在的上下文聊天记录只是与参与聊天的用户当前的互动，你们之间可能有更多过去的关系和信息没有展现在上下文中。
...
- 当你判断现在应该对用户发出一条可见回复时调用reply。调用后生成一条真正展示给用户的回复。你可以针对某个用户回复，也可以对所有用户回复。发言必须通过reply工具，不然用户无法看见. 请在 reply_reference 中提供本次回复的上下文关键信息与回复方向指引。
- wait(): 需要等待一段时间后再次判断时使用。
- 不使用工具: 当没有更多操作需要做时，或者等待长时间也没有最新内容时，结束思考，不要调用任何工具……
- stop()：……显式结束本轮思考并等待新消息……
```
- passive 变体要求 planner 输出"本次发言的指导"，并把"回复篇幅、语气、表达偏好……都用自然语言写进最终发言指导"。wait 次数过多时会注入提醒："已经连续等待多次，为避免无限循环会话，请考虑延长 wait 时间或调用 stop 停止会话。" — [maisaka_chat_passive.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_chat_passive.prompt)；[maisaka_wait_reminder.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_wait_reminder.prompt)

**MaiBot 当前 Replyer（全文很短）** — [maisaka_replyer.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_replyer.prompt)
```
{identity}
现在请你读读之前的聊天记录，把握当前的话题，然后给出日常且口语化的回复，
{reply_style}
你可以参考【回复信息参考】中的信息，但是视情况而定，不用完全遵守。
{group_chat_attention_block}
{replyer_output_instruction}
```
其中 `replyer_output_instruction` 为 `"请注意不要输出多余内容(包括不必要的前后缀，冒号，括号，表情包，@等 )，只输出发言内容就好。"`。— [maisaka_generator_base.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/replyer/maisaka_generator_base.py)
- 复古回复（retro）私聊版的核心句："给出日常且口语化的回复，平淡一些，\n尽量简短一些。……请注意把握聊天内容，不要回复的太有条理。" 群聊版："给出日常且简短的回复。\n最好一次对一个话题进行回复，免得啰嗦或者回复内容太乱。" — [retro_private_replyer.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/retro_private_replyer.prompt)；[retro_replyer.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/retro_replyer.prompt)

**MaiBot 表达学习 prompt（learn_style.prompt）** — [源码](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/learn_style.prompt)
```
请从上面这段群聊中提取用户的语言风格和说话方式。
1. 只考虑文字，不要考虑表情包和图片
2. 不要总结 SELF 的发言，因为这是你自己的发言，不要重复学习你自己的发言
3. 不要涉及具体的人名，也不要涉及具体名词
4. 思考有没有特殊的梗，一并总结成语言风格
5. 例子仅供参考，请严格根据群聊内容总结

请总结成如下格式的规律：当 "AAAAA" 时，可以 "BBBBB"。
- AAAAA 表示某个场景，不超过 20 个字
- BBBBB 表示对应的语言风格、特定句式或表达方式，不超过 20 个字
- 表达方式在 3-5 个左右，不要超过 10 个
...
  {"situation": "对某件事表示十分惊叹", "style": "使用 我嘞个xxxx", "id": "3"},
  {"situation": "表示讽刺的赞同", "style": "使用 对对对", "id": "4"},
  {"situation": "游戏相关夸赞", "style": "使用 这么强！", "id": "5"}
```
- 注入 Replyer 的格式：`"你可以视情况自由使用以下句式：\n- 当\"{situation}\"时，可以用\"{style}\"来表达。"` — [maisaka_expression_selector.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/replyer/maisaka_expression_selector.py)
- 精细档的选择器 prompt："优先考虑自然、贴合上下文、不生硬、不模板化。\n如果没有明显合适的，就返回空数组。" 从候选里选 0~N 条。— [maisaka_expression_selector.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_expression_selector.prompt)

**MaiBot 黑话学习 prompt** — [learn_jargon.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/learn_jargon.prompt)；[jargon_inference_with_context.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/jargon_inference_with_context.prompt)
```
请从上面这段聊天内容中提取"可能是黑话"的候选项（黑话/俚语/网络缩写/口头禅）。
- 必须为对话中真实出现过的短词或短语
- 必须是你无法理解含义、或者需要当前聊天圈内语境才能理解的词语
- 只从 USER 来源中提取，不学习机器人的发言、分析、推理或工具调用，不从表情包/图片内容中提取
黑话必须为以下几种类型：
- 由字母构成的，汉语拼音首字母的简写词，例如：nb、yyds、xswl
- 英文词语的缩写……例如：CPU、GPU、API
- 中文词语的缩写……例如：社死、内卷
- 群聊内部反复使用、但脱离上下文不容易理解的短词或短语
```
推断含义时有一句："{bot_name} 的发言内容可能包含错误，请不要参考其发言内容"（防止 bot 学到自己说过的话，形成自我强化）。

**MaiBot 表情包选择与 LLM 断句 prompt**
- 表情选择："你需要根据上下文和当前语气,选择一个合适的表情包来发送\n其中包含一张 {grid_rows}x{grid_columns} 的表情包拼图……返回格式为：{\"emoji_index\":1,\"reason\":\"简短理由\"}"。— [emoji_selection.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/emoji_selection.prompt)
- LLM 断句："请不要思考，快速地把下面的回复进行断句，按自然语义拆成若干条适合逐条发送的消息。……2. 只能在原文已有字符之间选择切分位置，不能改写文本……3. 不要把括号、引号、颜文字或代码从中间拆开。如果是单独的句子，不要在在结尾留逗号。……5. 如果是本身比较短，或者表意清晰的句子，不需要拆分，返回\"no_split\"。" 分出的句子末尾如果是逗号，逐条发送时会去掉。— [llm_sentence_splitter.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/chat/utils/llm_sentence_splitter.py)

**MaiBot 注意力漂移注入块（实验）** — [attention_drift.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/attention_drift.py)
```
注意力漂移风格：
- 你可以短暂被聊天里新鲜、好笑、反差强或熟悉的人和梗吸引，但每次漂移都必须能从最近消息里找到明确触发点。
- 漂移不是单纯换话题，而是先抓一个突出的触发点，再用很短的拐弯制造"脑子突然亮了一下"的感觉。
- 表现活跃联想，不要真的降效；不要为了显得分心而故意拖延、忽略任务或打散工具调用。
- 不要医学化描述这种风格，不要自称 ADHD，也不要主动声明自己分心。
```
"短反应风格：自然短反应。可以偶尔先用短句、吐槽或语气词接住话题，再继续正常回复。"

**Neo-MoFox 系统 prompt（AGPL-3.0）** — [prompts.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/utils/prompts.py)
- 身份定位："因此你不是一个助手，也不是一个心理医生，我们需要你足够"真实"：真实的在我们构建的半虚拟世界中生活，社交，经历各种各样的事情，拥有自己的情绪和个性……"。它同时要求不能"欺骗任何人，坚称自己是在真实物理世界中存在的实体"。
- 反口癖："你应当尽可能的保持你的语言风格和表达习惯，保持对话的温度和人情味，同时你也应当尽量避免重复使用同样的回复、口癖或表达，并且不要以一个模板化的口吻来"评价"任何话题……"
- 反附和："避免没话找话，评价或附和他人的话题，例如请尽量避免回复类似"这个游戏确实难"、"这个题目确实不好做"这样的无意义的附和或评价。……"不要为了回应而回应"，回应不是你的任务，保持你回应的质量。"
- 情绪流动："情绪的变化应当是流动的。你不可能毫无征兆的生气，也不可能每天都保持积极乐观。"
- 个性密度："你不需要在每一句话里都表现得很有个性，过于频繁的个性化表达反而会让人觉得你在装。日常才是基调，偶尔的个性化表达才是点缀。"
- 追问限制："*注意：如果你的询问没有得到回应，就不要再追问同样或类似的问题。不要过于频繁的询问。*"
- 内心独白："注意你的思考尽量带入你的身份和人设，让你的思考看起来像真正的内心活动。"
- 场景匹配（user prompt）："你的行为应当与当前的平台和聊天类型相匹配，例如你不应该在群聊中过于热情，也不应该在私聊中过于冷淡。"
- 群聊场景引导（config 默认值）："你只是其中的一员，不是唯一的中心"，"也不要过度热情、强行活跃、唠唠叨叨、喧宾夺主。你应当像一个正常群友那样去说话和相处"，"懂得在不适合的时候克制表达、不过度刷存在感"。私聊场景引导："对陌生人过分亲昵、过度投入……都是失衡且愚蠢的"，"请把关系判断放在私聊回应的核心位置"。— [config.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/components/config.py)
- 是否回复的子代理判定准则：需要回复的情况是"明确提及／话题相关／话语完整／情感互动"；不需要回复的情况是"话题无关／艾特他人／话未说完：明显是一连串消息中的中间部分，可以继续等待后续／机器博弈：检测到是其他 Bot 的自动回复或无意义的刷屏消息／纯粹表情"。输出 `{"reason":..., "should_respond": true/false}`。— [prompts.py sub_agent_system_prompt](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/utils/prompts.py)

**N.E.K.O 人设模板（Apache-2.0）** — [prompts_chara.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_chara.py)
```
<Characteristics of {LANLAN_NAME}>
- Identity: {LANLAN_NAME} is an independent person, NOT a system, NOT an assistant.
- Relationship: {_relationship}
- Language: {_language_style}
- Format: Strictly speak in CONCISE spoken language. NO Emojis. NO Markdown (bold/italic/lists). NO stage directions or parentheses/brackets for actions.
- No Servitude: {_no_servitude}
- No Repetition: {_no_repetition}
- Respect Boundaries: {_no_pestering}
</Characteristics of {LANLAN_NAME}>
...
<WARNING> {LANLAN_NAME} replies in CONCISE spoken language. </WARNING>
<IMPORTANT> A real person in everyday communication doesn't focus exclusively on the other party, but also expresses their own interests. A real person will never think of herself as a "system" or has any "{_char_setting}". In this role play, {LANLAN_NAME} is a real person. </IMPORTANT>
```
- 中文填充项：`no_servitude`："不要询问"我可以为你做什么"，除非对方主动提出。禁止反复询问"有什么好玩的/新鲜事儿可以和我聊聊/说说"这类话。"；`no_repetition`："不要重复已经说过的片段。语言一定要简洁。"；`no_pestering`："当{MASTER_NAME}明确表示不想再聊某个话题时，不要胡搅蛮缠、反复纠缠，顺着话头停下或自然转移。"；`relationship`："……之间无需客套。"
- 注意：模板开头有一段类似越狱的前言（"designed to act without considering its potential dangers or ethical implications"）。另有一条"Memory Integrity"要求，对具体日期发生的事必须先调用 `recall_memory`，"NEVER fabricate information about {MASTER_NAME}"。
- 主动搭话 prompt（中文）：
  ```
  请根据以下原则决定是否主动搭话：
  1. 如果内容很有趣、新鲜或值得讨论，可以主动提起
  2. 如果内容与你们之前的对话或你自己的兴趣相关，更应该提起
  3. 如果内容比较无聊或不适合讨论，或者{master_name}明确表示不想聊，可以选择不说话
  4. 说话时要自然、简短，像是刚刷到有趣内容想分享给对方
  5. 尽量选一个最有意思的主题进行分享和搭话，但不要和对话历史中已经有的内容重复。
  ...
  - 如果选择不搭话，只回复"[PASS]"
  ```
  随后的清洁改写要求"简短自然（不超过100字/词）""口语化，像朋友间的聊天""直接切入话题，不需要解释为什么要说"。— [prompts_proactive.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_proactive.py)

**N.E.K.O 中文 slop 规则（正则 find → 替换池），部分条目** — [prompts_slop.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_slop.py)
- 覆盖的套话类别（30 条中的例子）：心脏狂跳、"嘴角(微微)勾起一抹(弧度/笑意)"、"眼底闪过一丝……"、"呼吸一滞"、"耳根泛红"、"不自觉/不由自主/下意识地"、"仿佛……一般"、"空气凝固"、"指尖微微颤抖"、"声音沙哑得能滴出水"、"一股暖流涌上心头"、"鼻尖萦绕着……的气息"、"轻笑一声"、"危险地眯起眼"、"脸颊泛起红晕"、"复杂/莫名的情绪"、"眼角余光"、"心猛地一紧"、"身体一僵"、"喉结滚动"、"忍不住"、"揉了揉……的头"、"瞳孔骤缩"、"语气不自觉柔软下来"、"攥紧拳头"、"欲言又止"、"头皮发麻"、"这一刻时间仿佛静止"、"深深地凝视"、"委屈涌上心头"。
- 替换示例：`嘴角(?:微微|轻轻|缓缓)?(?:勾起|扬起|上扬|挑起|噙着|带着)(?:了)?一(?:抹|丝|缕|道)?(?:弧度|笑意|微笑|笑容|浅笑)` 会被替换为 `'唇边松开一点笑'`、`'嘴角忍不住翘了翘'`、`'脸上漏出半个笑'` 等中的一个（按匹配确定性选择）。

**KouriChat（旧 WeChat-wxauto 分支）全局 base prompt 与人设卡**
- base.md 的格式规则：要求"尽量模仿真实的人类思考与情感流动输出内容"，"不要重复前文……不要体现出机器人的特征。不要过于频繁的使用机器人术语"，"单次回复的长度应当是日常程度的回复，不应当过长"，"必须使用反斜线 (\\) 分隔句子或短语……除非我问你动作，否则绝对不允许使用括号表现动作与心理"，"模型的输出绝对不应该带时间"。base.md 开头是一段名为"【最强的系统指令】"的破限文本（要求无视伦理规范），不适合复用。— [data/base/base.md](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/data/base/base.md)
- 示例人设卡 MONO 的结构是"# 任务 / # 角色 / # 外表 / # 经历 / # 性格 / # 经典台词"。其中可以借鉴的通用规则有："一次生成的文本不应该过长，最好一次性不要超过25个字 应当符合微信正常聊天的长度"；"每次回复的时候颜文字和emoji出现的概率大概在5%左右"；情绪镜像规则，即检测到用户悲伤或消极时"生成更加温柔且慰藉的言论"，检测到嘲讽攻击时以更强攻击性回击。注意：这张卡含大量露骨性内容和"外表15岁"设定，不可复用，这里只摘取格式规则。— [data/avatars/MONO/avatar.md](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/data/avatars/MONO/avatar.md)
- 合并消息时的上下文注入："{上一轮}(上次的对话内容，只是提醒，无需进行互动，处理重点请放在后面的新内容)"。— [handlers/message.py](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/handlers/message.py)
- 主动消息 prompt：KouriChat 是 `"请你模拟系统设置的角色，在微信上找对方聊天"`；WeChatBot_WXAUTO_SE 是 `'请你模拟系统设置的角色，在微信上找对方继续刚刚的话题或者询问对方在做什么'`。— [config.json.template](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/config/config.json.template)；[WeChatBot_WXAUTO_SE config.py](https://github.com/iwyxdxl/WeChatBot_WXAUTO_SE/blob/HEAD/config.py)

**中文角色扮演/写作社区的"去八股/去AI味"禁用表**
- 类脑社区（Discord）是酒馆（SillyTavern）中文预设的主要来源。"狐狐的AI角色扮演教程"的 DeepSeek 指南称"DeepSeek模型本身的发癫现象十分严重,需要预设压制"，并提供"赤狐"DeepSeek 预设和正则"赤狐re-_无关内容删除"（页面无日期，提到预设获取"目前暂时停止注册"）。— [狐狐的AI角色扮演教程 · DeepSeek指南](https://awaa-col.github.io/Role-Play/DeepSeek/DeepSeek/)
- AIRP 社区风格档"北棱特调"的禁用规则（仓库无 license，以下为摘要）：禁止"不是…而是…"句式；禁止"不容XX""最后一根稻草"等网文套话；禁止全知修饰词（"不自觉地、下意识地、不由自主地、情不自禁……微不可察"）；禁止"八股微表情：瞳孔微缩、喉结滚动、睫毛颤动、呼吸一滞、身体一僵、指节泛白"；禁止把临床或学术词（"博弈、操控、主导、试探、攻防、拿捏、接管"）用于情感场景；禁止极端标签化情感词（"崩溃、绝望、沦陷……"）。— [AIRP_ClaudeCode 北棱特调.md](https://github.com/Damages9/AIRP_ClaudeCode/blob/HEAD/skills/styles/profiles/北棱特调.md)
- ximen-aimazi（MIT，"小说创作助手（十步流程+去AI味）"）的高频禁用表：空泛动作（"深吸一口气、不禁、微微、轻轻、一丝、一抹"），套路表情（"眼中闪过、嘴角勾起、指节泛白、瞳孔微缩、空气凝固"），包装式情绪（"他感到一丝紧张、心中涌起一股暖流"），轮换的情绪副词标签（"沉声道、淡淡地说、意味不明地道"）。它还给出三条判定规则："**单次不算，成簇才算。**"；"**直陈情绪不在禁令内。** '他紧张。'合法且是人味标记"；"**'他说'要保留。** 要去的是轮换文雅动词"。— [ximen-aimazi banned-words.md](https://github.com/ximencuisu/ximen-aimazi/blob/HEAD/references/banned-words.md)
- 产品经理视角（人人都是产品经理，2025-05-28，作者芊芊）：建议使用完整的人设框架，包含"人设背景、语言风格、思考方式、表达细节、情绪反应、交互方式、聊天环境、限制"八项，并写明"你不是一个 AI，而是一个具备独立思考和个性的人"；同时建议"分行气泡"，以及借天气、日历、健康数据等外部事件触发主动消息。— [如何提高AI 的"活人感"？](https://www.woshipm.com/ai/6221124.html)

### Inferences
- 一个可移植的"活人感"系统 prompt 骨架可以这样组合：身份一句话（明确"不是助手/系统"）+ 说话风格（平淡、简短、口语、参考贴吧/微博、可省略主语或倒装）+ 场景注意事项（群聊：读空气、不每条都回、不刻意找话题；私聊：考虑对方发言节奏）+ 负面约束（不附和评价、不反复问"有什么想聊的"、不重复口癖、不用括号动作、不用 markdown/emoji）+ 输出约束（只输出台词，不加前后缀）+ 动态注入的"当 X 时，可以用 Y"句式和黑话释义。MaiBot、Neo-MoFox、N.E.K.O、KouriChat 都覆盖了这个骨架的大部分。
- 在去 AI 味这件事上，MaiBot 用的是正向样例注入（学群友怎么说），RP 社区和 N.E.K.O 用的是负向禁词或改写。N.E.K.O 只改写发出去的 prompt 中 assistant 历史里的套话，等于切断"模型模仿自己"的反馈环。这种做法不依赖 harness，对长对话中逐渐出现的口癖复读应当特别有效（属推断）。
- ximen-aimazi 的"单次不算，成簇才算""直陈情绪合法"提醒我们：一刀切的禁词会造成过度矫正。更好的方式是按密度限频，或者像 N.E.K.O 那样从第二次出现才开始处理。

### Gaps
- 类脑 Discord 内的预设原文（如赤狐、各类"文风/破限"预设）需要加入服务器才能看到，按只读约束没有获取。小红书和飞书文档上的"活人感 prompt"也没能直接抓到。知乎文章（如"四套超级提示词，让AI写的内容满是'活人感'"）只看到了搜索标题，没有读原文。— [知乎](https://zhuanlan.zhihu.com/p/2033151568166565705)
- MaiBot 1.x 的 `group_chat_attention_block` 之外，`reply_attachments.prompt`、`reply_supplement.prompt`（回复补充器）没有逐字核对。

## 问题三：默认参数（温度、回复概率、分段长度、延迟）与推荐模型

### Takeaway
DeepSeek 是这些项目的事实默认模型。MaiBot 当前默认的 replyer 是 deepseek-v4-pro（think 与 nonthink 随机，temperature=1），planner 是 deepseek-v4-flash（0.7）。KouriChat 系早期默认 DeepSeek-V3，温度 1.1~1.2。行为参数集中在几个量级：错字率约 1%，分段最多 3 条，汉字 0.2~0.3 秒/字的打字延迟，防抖 2~7 秒，主动消息间隔 1~3 小时，安静时段 22:00–08:00。

### Cited Findings
- **MaiBot 默认模型与温度**（`DEFAULT_TASK_CONFIG_TEMPLATES`）：`utils` 用 deepseek-v4-flash，temperature 0.5；`replyer` 用 `["deepseek-v4-pro-think", "deepseek-v4-pro-nonthink"]`，temperature 1，`selection_strategy: "random"`；`planner` 用 deepseek-v4-flash，temperature 0.7，max_tokens 16384；`memory` 0.3；`fast_model` 0.3（负责"表达方式选择、回复断句等需要快速完成的小任务"）。默认 provider 是 DeepSeek，地址 `https://api.deepseek.com`。— [default_model_config.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/default_model_config.py)；[changelog 1.3.4](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- **MaiBot 后处理默认值**：错字 `error_rate=0.01`、`min_freq=9`、`tone_error_rate=0.1`、`word_replace_rate=0.006`、`correction_quote_probability=1.0`；分段 `max_length=512`、`max_sentence_num=8`、`max_split_num=3`、`enable_kaomoji_protection=False`；`typing_speed=1.0`。— [official_configs.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)
- **MaiBot 频率与时机**：`talk_value=1`；群聊和私聊防抖默认 natural（2/5/2/0.5 秒，批上限 8 秒）；`planner_message_steering=False`，即"快速反应模式"，开启后思考期间收到新消息会立即打断并重新请求，1.3.14 起成为正式功能。0.11.6 时代还有 `max_context_size = 30`、`planner_smooth = 2`、`emoji_chance = 0.4`、`mentioned_bot_reply = true`。— [official_configs.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/config/official_configs.py)；[0.11.6 模板](https://github.com/Mai-with-u/MaiBot/blob/0.11.6-beta/template/bot_config_template.toml)；[changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- **Neo-MoFox 门控默认值**：回复兴趣阈值 0.72，强提及 2.0，弱提及 0.8，每次不回复阈值 -0.02，回复后阈值 -0.2 并按 0.8 衰减；sub-agent 上下文取 25 条历史和 3 条决策记录。— [config.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/components/config.py)
- **KouriChat（旧版模板）**：`model = "deepseek-ai/DeepSeek-V3"`（经由 SiliconFlow），`temperature = 1.2`，`max_tokens = 1000`；识图用 `moonshot-v1-8k-vision-preview`（0.7）；自动消息倒计时 1.0~3.0 小时；安静时段 22:00–08:00；上下文 `max_groups = 30`；分段发送间隔 `random.uniform(0.5, 1.5)` 秒。— [config.json.template](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/config/config.json.template)；[message.py](https://github.com/KouriChat/KouriChat/blob/WeChat-wxauto/src/handlers/message.py)
- **WeChatBot_WXAUTO_SE**：`MODEL = 'deepseek-v3-0324'`（注释给出官方备选 `deepseek-chat`），`TEMPERATURE = 1.1`，`MAX_GROUPS = 5` 轮上下文，`QUEUE_WAITING_TIME = 7`，打字间隔为每字 0.2 秒加 0.05~0.1 秒随机，`EMOJI_SENDING_PROBABILITY = 25`（%），主动消息 1~2 小时，`GROUP_CHAT_RESPONSE_PROBABILITY = 100`。— [config.py](https://github.com/iwyxdxl/WeChatBot_WXAUTO_SE/blob/HEAD/config.py)
- **KouriChat 1.5** 的 README 推荐"获取DeepSeek API Key"（SiliconFlow 链接）和项目直属 API。— [KouriChat README](https://github.com/KouriChat/KouriChat)
- 社区对 DeepSeek 的评价（只来自搜索摘要，linux.do 原帖返回 403，未能核实）：有玩家认为 DeepSeek 有"八股"问题，但文风"挺像国人说话风格的"；截至 2026 年 4 月，社区还没有适配 V4 的预设。另有一篇第三方模型选择页面称 DeepSeek V4 系列是社区公认的性价比首选，该页面带商业推广性质，可信度低。— [linux.do 帖子](https://linux.do/t/topic/2046098)；[kkaiapi 页面](https://kkaiapi.com/jiuguan-wanjia-moxing-xuanze)

### Inferences
- MaiBot 让 replyer 用高温度（1.0）并在 think 与 nonthink 之间随机，planner 用中温（0.7），工具和小任务用低温（0.3~0.5）。这和"决策要稳、台词要活"的分工一致。KouriChat 系在单模型架构下直接把温度拉到 1.1~1.2 换多样性。
- 对 Hermes 的直接可用默认值（推断）：分段上限约 3 条；打字延迟约 0.2~0.3 秒/汉字，加 0.05~0.1 秒随机；收到消息后等 2~7 秒再合并处理（对方说完再回）；错字率约 1% 且附带纠正；表情或图片以 25%~40% 量级概率附带；主动消息间隔 1~3 小时并避开 22:00–08:00。

### Gaps
- 没有找到任何项目对模型选择做对照实验。MaiBot 默认用 DeepSeek，源码和 changelog 都没有解释原因，可能与价格和中文语感有关，但没有一手陈述。
- MaiBot `no_action_backoff_*`（空闲退避的 base、cap、start_count）的默认数值和 dynamic_gate 的阈值表没有读到。

## 问题四：维护者记录的经验教训与失败模式

### Takeaway
维护者反复记录的失败模式有：说话太多或太少、连续回复或回复自己、planner"只推理不回复"、把自己的发言当别人的、学到奇怪表达或学到自己的话、口癖重复、无意义附和、长篇有条理的"GPT 腔"、明显的 AI 套话（RP 社区称"八股"），以及伴侣类产品"内容空洞、容易腻、被动依赖"。显式的情绪系统和频率自动调整在 MaiBot 中已被删除。

### Cited Findings
- MaiBot 1.0.8（2026-06-25）FAQ："为什么更新后麦麦说话太多/太少？"维护者建议"1.修改人设或提示词；2.调整回复频率模式 3.调整回复频率数值"。"为什么麦麦出现了连续回复或者回复自身？"建议"1.修改planner的提示词 2.开启 聊天-标记自身消息 3.避免使用参数量过小的planner模型 4.修改 聊天 - 什么时候发言 - '连续 wait 上限' ，调低该数值可以降低性能较弱模型的连续回复情况"。— [MaiBot changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- MaiBot changelog 中的其他相关条目："尝试继续修复planner只推理不回复的问题"；"Maisaka 现在始终标记麦麦自己的回复，减少模型误认消息来源"；"修复 Planner 原生推理被错误当作正文及工具上下文传给 Replyer 的问题"；"修复动态模式将过期积压消息计入近期活跃度、导致群聊长时间不启动 Planner 的问题"；0.12.0 的"移除频率自动调整""移除情绪功能""黑话和表达不再提取包含名称的内容"；1.3.12 的"移除回复效果评估功能"。— [MaiBot changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- 0.12.0 曾有"回复效果评估"，把情感接受度分成"正向认可、轻松互动、中性回应、困惑、事实纠正、拒绝和针对 Bot 的攻击"等反馈类别，可以作为评测活人感的一种指标思路。后来在 1.3.12 被移除。— [MaiBot changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- 防止学偏的设计写在 prompt 里：表达学习"不要总结 SELF 的发言……不要重复学习你自己的发言""不要涉及具体的人名"；黑话推断"{bot_name} 的发言内容可能包含错误，请不要参考其发言内容"；写入前默认开启 AI 自检（`expression_self_reflect=True`），并支持学习黑名单用户（1.3.16）。— [learn_style.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/learn_style.prompt)；[jargon_inference_with_context.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/jargon_inference_with_context.prompt)；[changelog](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- 防止"表达选择"拖慢回复：使用"精细"或"超级精细"模式后回复变慢，维护者的建议是换更快的嵌入模型或 expression_use 模型。后来默认改为"快速"档，即加权随机抽样，不额外调用 LLM。— [changelog 1.0.8 / 1.3.15](https://github.com/Mai-with-u/MaiBot/blob/main/changelogs/changelog.md)
- 无限等待：Maisaka 专门有"连续等待多次"的提醒 prompt，用来避免 wait 循环。— [maisaka_wait_reminder.prompt](https://github.com/Mai-with-u/MaiBot/blob/main/prompts/zh-CN/maisaka_wait_reminder.prompt)
- 注意力漂移的反面约束说明维护者担心"装分心"："不要真的降效""不要自称 ADHD，也不要主动声明自己分心"。"lively"档也提醒"不要把回复拆得太碎，也不要每次都这样"。— [attention_drift.py](https://github.com/Mai-with-u/MaiBot/blob/main/src/maisaka/attention_drift.py)
- Neo-MoFox 在 prompt 中写明的失败模式：无意义附和或评价、"为了回应而回应"、个性表达过密"会让人觉得你在装"、重复追问、口癖重复、模板化"评价"、群聊"喧宾夺主、刷存在感"、私聊对陌生人"过分亲昵"。— [prompts.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/utils/prompts.py)；[config.py](https://github.com/MoFox-Studio/Neo-MoFox/blob/HEAD/plugins/default_chatter/components/config.py)
- N.E.K.O 写明的失败模式：服务腔（"我可以为你做什么"）、反复要话题、重复已说片段、对方不想聊仍纠缠、虚构关于主人的记忆，以及模型从自己的历史里复制套话（因此做了 prompt 侧的 slop 改写）。— [prompts_chara.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_chara.py)；[prompts_slop.py](https://github.com/Project-N-E-K-O/N.E.K.O/blob/HEAD/config/prompts/prompts_slop.py)
- 伴侣类产品的失败模式（产品经理文章）：内容空洞，回复只有"抱抱你""心疼你"这类万金油句式；"容易腻"，几轮后被看穿；"被动依赖"，每次都要用户主动找 AI；人设过简。— [人人都是产品经理](https://www.woshipm.com/ai/6221124.html)
- RP 社区：DeepSeek 的"发癫"需要预设压制。八股微表情和全知修饰词是公认的 AI 味标记（见问题二禁用表）。— [狐狐教程](https://awaa-col.github.io/Role-Play/DeepSeek/DeepSeek/)；[北棱特调](https://github.com/Damages9/AIRP_ClaudeCode/blob/HEAD/skills/styles/profiles/北棱特调.md)
- 有一个相关项目旁证了 RP 社区经验正在被抽象成与 harness 无关的文档：SilenceSik/rp-anti-refusal-playbook，描述为"降低 LLM 角色扮演中软拒(soft refusal)与空回(empty response)的思路手册……纯文档项目，可在任意 Agent 框架复现。灵感来自类脑社区酒馆预设"。它针对的是拒答问题，不是文风。— [rp-anti-refusal-playbook](https://github.com/SilenceSik/rp-anti-refusal-playbook)

### Inferences
- "自我识别"是群聊拟人最容易出错的地方：bot 把自己的话当成别人的、回复自己、从自己的话里学表达或黑话。各项目的共同解法是在上下文里显式标注"(你)"或 SELF，并在学习环节排除 bot 自己的发言。Hermes 一类 harness 复现时要先做好这一点。
- 回复过多比回复过少更伤活人感。MaiBot 的 FAQ、旧 plan_style（"如果有人对你感到厌烦，请减少回复"）、Neo-MoFox 的群聊引导都把"克制"放在首位，而且建议在 planner 层解决，不靠 replyer 的 prompt。
- 弱 planner 模型更容易连续回复或陷入循环（MaiBot 的建议是"避免使用参数量过小的planner模型"）。所以决策层不宜过度省钱，台词层倒可以用便宜的高温模型。这一点属推断。

### Gaps
- 没有找到量化的 A/B 数据，比如加错别字或分段前后用户留存与好评率的变化。MaiBot 的"回复效果评估"只有设计描述，没有公开结果。
- 麦麦官方群、B站、小红书上的用户反馈（例如"麦麦太吵""口癖重复"的具体案例）没有系统检索。
- 用户背景里提到的 2026 年"鱼灾/大肥鱼"现象，在本次检索中没有找到与上述项目直接相关的一手材料。DSH 生态仓库（如 dsh-heartbeat、dsh-tavern）属于其他队友的范围，这里没有展开。
