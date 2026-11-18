---
layout: default
---

# Engineering Notes

Field notes on the failures that never throw an error: incident archives, test rot, multimodal fallbacks, character consistency, group-chat pacing, and GUI automation.

## 中文

- [01. 14 个不报错的失败](01-silent-failures.md) — 14 个静默故障，没有一个抛异常
- [02. 测试的静默腐烂](02-test-rot.md) — 桩漂移、吞异常与覆盖盲区
- [03. 多模态降级的六个决策](03-multimodal-fallback.md) — 什么时候等，什么时候放弃
- [04. 把「不像」变成数字](04-character-consistency.md) — 角色一致性的量化验收
- [05. 群聊机器人：一次讨论只回一条](05-group-burst.md) — 把消息聚合回事件
- [06. GUI 自动化的可靠性](06-gui-automation.md) — 为什么「发送成功」不可信
- [07. 记忆的写入端：什么该存，记到谁头上](07-memory-write-time.md) — 五条写入判据与两种主语错位
- [08. 心跳还在，消息不来：推送通道的静默故障](08-push-silence.md) — 连接存活不等于业务活跃
- [09. 同一批媒体，为什么回了两条：并发下的「占位」](09-concurrent-media.md) — 只读的闸门挡不住并发
- [10. 引用了别人的图，为什么说看不到：跨人媒体的认领](10-quoted-media.md) — 防串台的规则会误伤显式引用
- [11. 群聊里，什么时候该说话：主动发言的决策](11-proactive-speech.md) — 概率决定冒泡，结构决定像不像群里的人
- [12. 记忆的三层：原话、事实与摘要](12-memory-layers.md) — 每层要能回答上一层答不了的问题
- [13. 群黑话表：让机器人听懂「每个字都认识，连起来不懂」](13-group-glossary.md) — 黑话是群内约定，不是生词

## English

- [01. 14 Failures That Never Threw an Error](en/01-silent-failures.md) — fourteen silent failures, none of them exceptions
- [02. How Tests Rot Quietly](en/02-test-rot.md) — stub drift, swallowed exceptions, and coverage blind spots
- [03. Six Fallback Decisions for Multimodal Chatbots](en/03-multimodal-fallback.md) — when to wait, when to give up
- [04. Making "It Doesn't Look Like the Character" Measurable](en/04-character-consistency.md) — quantifying character consistency
- [05. One Topic, One Reply](en/05-group-burst.md) — treating messages as a conversation
- [06. Why "Sent Successfully" Means Nothing in GUI Automation](en/06-gui-automation.md) — the three defenses behind reliable sending
- [07. The Write Side of Memory: What to Store, and Whose Fact It Is](en/07-memory-write-time.md) — five write rules and two attribution failures
- [08. The Heartbeat Is Alive, the Messages Are Not](en/08-push-silence.md) — liveness is not activity
- [09. Why One Batch Produced Two Replies](en/09-concurrent-media.md) — a read-only gate cannot stop concurrency
- [10. Quoting Someone Else's Image: Who Claims the Media?](en/10-quoted-media.md) — anti-mixup rules hurt explicit quotes
- [11. When to Speak in a Group Chat: Deciding to Join](en/11-proactive-speech.md) — probability pops up; structure belongs
- [12. Three Layers of Memory: Verbatim, Facts, and Summaries](en/12-memory-layers.md) — each layer must answer what the previous cannot
- [13. A Group Glossary: Understanding Slang You Can Read but Not Parse](en/13-group-glossary.md) — slang is a group-local convention

## Note on the numbers

All timestamps, counts, ratios, and costs in these articles are examples drawn from actual operation. They are not universal benchmarks. Identities, platforms, and paths have been redacted.

## License

© 2026 Engineering Notes contributors. The articles are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](https://github.com/AirXsLink/engineering-notes/blob/main/LICENSE) for the full text.
