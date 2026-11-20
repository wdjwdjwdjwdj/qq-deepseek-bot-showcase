# QQ-DeepSeek-Bot（大肥鱼）— 展示版

> 一个跑在 QQ 上的「数字生命」型 AI 机器人：基于 MaiBot + NapCat + 多模型 LLM 路由，具备长期记忆、情绪与认知状态、关系亲密度演化、定时任务、QQ 空间社交和语音能力，在服务器上 7×24 实际运行。

本仓库是**脱敏展示版**：仅收录自研核心代码，所有配置文件、运行数据、人格/调研文档均已移除，代码中的账号标识已替换为占位示例（10001–10007），不含任何真实密钥、真实 QQ 号或聊天数据。生产版仓库保持私有。

## 架构概览

```
QQ 客户端 ──NapCat（OneBot v11 HTTP :3002）── MaiBot（对话主脑 / Planner / 工具调用）
                                                    │
        ┌───────────────────┬──────────────────────┼──────────────────┐
   自研计划任务脚本（src/）   MaiBot 自研插件（src/plugins/）   QQ 空间桥接（onebot-qzone :5700）
   认知 / 记忆 / 生活 / 社交      定时管家 / 关系账本 / 语音        动态浏览 / 点赞 / AI 评论
                                                    │
        LLM 路由（botcommon）：免费池优先（智谱 / 商汤 / OpenRouter 免费模型）
                              → 付费 DeepSeek 兜底，token_guard 做余额与日预算门控
```

- **接入层**：NapCat 提供 OneBot v11 HTTP API，所有外呼走 `botcommon` 的域名白名单 HTTP 客户端，禁止重定向、只允许固定目标（本机 NapCat、官方 LLM API），从代码层收死网络边界。
- **对话层**：MaiBot 负责对话与工具调用；自研插件以 LLM 工具形式注册进对话，让机器人在聊天中就能建定时任务、查关系分、发语音。
- **状态层**：一组由系统计划任务周期调用的 Python 脚本，分别维护情绪/认知状态、上下文记忆归档、按人隔离的长期记忆、好友亲密度账本、每日生活时间线，形成「越聊越像一个具体的人」的演化闭环。
- **成本层**：免费模型池优先、付费模型兜底；`token_guard` 定时查 DeepSeek 余额并按日预算硬顶熔断，超支自动降级并告警。

## 核心模块

| 文件 | 作用 |
|---|---|
| `src/botcommon.py` | 公共基础设施：白名单 HTTP、provider key 读取、JSON 原子写、NapCat 发送、免费/付费 LLM 路由 |
| `src/cognition.py` | 认知引擎：情绪（PAD）增量、意图与矛盾检测、技能卡，周期性更新机器人认知状态 |
| `src/context_memory.py` | 上下文记忆归档：按半天阈值把聊天记录压缩成 Markdown 长期记忆 |
| `src/person_memory.py` | 按人记忆隔离：给上游检索补 fail-closed 过滤，防止不同人的记忆串扰 |
| `src/friends_ledger.py` | 关系账本：每位群友的亲密度分数与档位，每日复盘后由 LLM 自行调分 |
| `src/daily_life.py` | 生活引擎：结合真实天气与昨日高光生成「机器人的一天」时间线与朋友圈草稿 |
| `src/evolve_loop.py` | 空闲自进化 loop：闲时读记忆与状态产出改进提案（提案制，不自动改人设） |
| `src/token_guard.py` | 余额守护与预算门控：查 DeepSeek 余额、日预算硬顶、超支熔断与告警 |
| `src/qzone_social.py` | QQ 空间社交引擎：刷好友动态、点赞、AI 评论 |
| `src/scheduled_sender.py` | 定时消息发送器：按时间表经 NapCat 给指定人/群发消息 |
| `src/welcome_bot.py` | 新人入群欢迎：监测群成员变化，LLM 现写欢迎语并 @ 新人 |
| `src/noreply_watch.py` | 「进消息不回复」哑火告警：扫日志检测 Planner 无响应并私聊告警 |
| `src/night_watch.py` | 夜间兜底监控：NapCat 会话与关键端口的只读体检 |
| `src/voice_tts.py` | 语音合成：edge-tts 合成中文语音，经 NapCat 发 QQ 语音消息 |
| `src/plugins/SchedulerManager` | MaiBot 插件：在对话中创建/查看/删除定时消息任务 |
| `src/plugins/FriendLedger` | MaiBot 插件：在对话中查询与群友的亲密度（关系账本） |
| `src/plugins/VoiceSpeak` | MaiBot 插件：在对话中触发语音消息发送 |

## 特性

- **多模型路由 + 成本门控**：免费池优先、付费兜底，余额查询与日预算硬顶熔断，避免 token 烧穿。
- **长期记忆三件套**：上下文归档压缩、按人隔离检索（fail-closed）、关系亲密度账本，记忆不串人。
- **拟人状态演化**：情绪 PAD 状态、每日生活时间线、空闲自进化提案，让行为随互动持续变化。
- **主动社交**：QQ 空间动态互动、新人欢迎、定时消息，机器人不只被动应答。
- **可观测与自愈**：哑火检测、夜间体检、余额告警，异常第一时间私聊通知维护者。
- **网络边界收死**：所有外呼经域名白名单客户端，只允许本机服务与官方 API，防 SSRF/重定向劫持。

## 技术栈

Python 3（标准库为主）· MaiBot · NapCat（OneBot v11）· DeepSeek API · 智谱 GLM / 商汤 / OpenRouter 免费模型池 · edge-tts · SQLite · Windows 计划任务

## 说明

- 本展示版不含部署配置与运行数据，无法直接运行；`model_config.toml`、`.env` 等密钥文件按 `.gitignore` 规则排除，所需环境变量见 `.env.example`（仅键名占位）。
- 代码中的 QQ 号均为占位示例（10001–10007），群号、好友种子分数等示例数据已经脱敏处理。
