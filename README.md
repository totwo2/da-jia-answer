# da-jia-answer — Everyone Answers

> Ask one question to multiple LLMs at once. Auto-discovers configured models, you pick which to query, answers come back side-by-side.

[中文说明见下方](#中文说明)

---

## What it does

1. **Auto-discovers** all LLMs configured on your system (via LLM-assisted path discovery)
2. **Lists models** for you to select which ones to compare
3. **Calls them in parallel** using OpenAI-compatible API
4. **Shows answers side-by-side** in a multi-column comparison

Zero manual configuration — just pick models and ask. Works with any OpenAI-compatible endpoint (OpenAI, DashScope, Ollama, Hunyuan, One API, etc.).

## Install

```bash
# From GitHub Agent Skills
gh skill install {owner} da-jia-answer

# From SkillHub (CN)
skillhub install da-jia-dou-lai-hui-da --namespace user_c18b02ff
```

## Usage

```
Everyone answers: What's the monthly payment for a 1M loan, 30 years, 3.02% rate?
Compare models: Best practices for React state management?
大家都来回答：贷款100万，30年，利率3.02%，等额本金，每月还多少？
```

## Security

- Config files are **read-only** — never modified
- API keys are **masked** in all outputs
- Keys never written to logs or skill files

## License

MIT

---

## 中文说明

# 大家都来回答 (da-jia-answer)

> 一个问题，同时问多个已配置的大模型，并行发问、多栏对比呈现各家回答。

## 这是什么

当你抛出一个问题时，主 Agent 会：

1. **自动探测**你系统上已配置的所有大模型（Discovery，由当前 LLM 通过内置提示词完成）
2. **列出模型**让你勾选想对比的几个
3. **并行发问**给选中的模型
4. **多栏对比**呈现各家回答

全程你只需要"勾选模型"，无需手动填写任何 base_url / api_key。

## 触发方式

对着助手说以下任一句话即可激活：

- 大家都来回答
- 三路对比
- 多个 AI 都回答
- 对比回答
- 大家怎么说
- 你们几个怎么看

例如：`大家都来回答：贷款100万，30年，利率3.02%，等额本金，每月还多少？`

## 工作原理

```
问题 → Discovery(LLM给配置路径) → 读配置 → 解析模型列表
     → 用户勾选 → 并行 OpenAI 兼容 API 调用 → 多栏对比输出
```

调用采用 OpenAI 兼容协议，兼容任何提供 `/chat/completions` 接口的模型服务
（DashScope / OpenAI / Ollama / 混元 / One API 等）。

## 安全说明

- **配置文件仅读不写**，不修改你的任何配置
- **API Key 不出现在回复里**，展示时用 `***` 脱敏
- 不将 Key 写入任何日志或 Skill 文件

## 目录结构

```
da-jia-dou-lai-hui-da/
├── SKILL.md                         # 主文件：触发词 + 6 步流程 + 错误处理 + 安全约束
├── README.md                        # 本文件
└── assets/templates/
    └── compare_result.md            # 多栏对比输出模板
```

## 版本

- **v1.0.0** — 初始发布。通用模型对比方案（LLM 自报配置路径 → 读配置 → 勾选 → 并行调用 → 对比），已端到端实测通过。

## 使用前提

系统上已配置至少一个可用的大模型（有有效的 base_url + api_key + model）。
若探测不到，Skill 会提示你手动提供。
