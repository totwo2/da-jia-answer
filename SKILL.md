---
name: da-jia-answer
slug: da-jia-answer
version: 1.0.1
displayName: Everyone Answers
description: |
  Ask one question to multiple configured LLMs simultaneously. Auto-discovers models on the user's system, lets them pick which to query, then calls them in parallel and presents answers side-by-side for comparison. Zero manual configuration — just select models and ask.
  Triggers: everyone answers, multi-model compare, ask all models, A/B/C test, side-by-side LLM, "da-jia-answer".
  一个问题同时问多个已配置的大模型，并行发问、多栏对比呈现各模型回答。主 Agent 自动探测用户系统上已配置的大模型，列出让用户勾选，然后并行调用、多栏对比。全程用户只需勾选模型，无需手动配置。
  触发词：大家都来回答、三路对比、多个 AI 都回答、对比回答、大家怎么看、你们几个怎么看
license: MIT
tags:
  - multi-model
  - comparison
  - llm
  - parallel
  - benchmark
allowed-tools: "Read Write Edit Bash Glob Grep WebFetch WebSearch Skill Agent"
---

# da-jia-answer — 多模型对比回答

> 一个问题，多个 AI 同时回答。并行发问、多栏对比，让用户一眼看到差异。

---

## 1. 触发词（命中即启动）

| 用户说法 | 意图 |
|---------|------|
| "大家都来回答" / "大家怎么看" / "你们几个怎么看" | 多模型对比 |
| "三路对比" / "多模型对比" / "对比回答" | 多模型对比 |
| "A/B/C 回答" / "不同 AI 回答对比" | 多模型对比 |
| "问所有模型" / "ask all models" | 多模型对比 |
| "side-by-side LLM" / "roast my project" | 多模型对比 |

**排除**：用户只是闲聊、没有明确的"希望多个 AI 回答"意图时，不触发。

---

## 2. 执行流程（Agent 操作手册）

### 2.1 总体流程

```
用户提出问题
  → 1. Discovery：探测用户系统上的大模型配置
  → 2. 展示模型列表，让用户勾选
  → 3. 并行调用选中的模型
  → 4. 格式化对比输出
  → 5. 向用户展示结果
```

### 2.2 步骤详解

#### 第 1 步：Discovery — 探测模型配置

**操作**：向当前运行的 LLM 注入以下提示词，获取用户系统的模型配置文件路径：

```
请输出你当前系统中所有已配置的大模型相关的配置文件绝对路径。
每行一个路径，只输出路径，不要任何解释说明。
常见位置包括但不限于：
- ~/.qclaw/ 下的配置文件
- ~/.config/ 下的模型配置文件
- ~/.openclaw/ 下的配置文件
- ~/.ollama/ 下的配置
- ~/.workbuddy/ 下的配置
- 环境变量中引用的配置文件路径
- 用户目录下的 .env 文件（包含 API Key 配置）
- One API / OpenWebUI 等统一代理的配置文件

如果找不到任何配置文件路径，请输出：NONE
```

**执行方式**：将该提示词作为下一轮 system 级别指令注入，LLM 回复后解析出路径列表。

**解析结果**：
- 如果返回 `NONE` 或无路径 → 提示用户"未找到任何配置文件，请确认是否已配置大模型"
- 如果返回路径列表 → 进入第 2 步

#### 第 2 步：并行读取配置文件

对第 1 步得到的每个路径，**并行**执行 `read`：

- 文件存在 → 读取内容
- 文件不存在 / 权限不足 → 跳过，记录为"不可读"
- .env 类文件 → 提取其中 `API_KEY`、`BASE_URL`、`MODEL` 等关键变量

**解析模型配置**：

| 文件类型 | 解析方式 |
|---------|---------|
| JSON（gateway.json 等） | 解析 `models[]` / `channels[]` 字段 |
| YAML | 解析 `channels:` / `models:` / `endpoints:` 段落 |
| .env | 提取 `BASE_URL_*`、`API_KEY_*`、`MODEL_*` 变量组 |

**统一模型对象格式**：
```python
{
    "id": str,        # 唯一标识，如 "model_1"
    "name": str,      # 模型显示名，如 "qwen-max"
    "base_url": str,  # 调用地址，如 "https://api.openai.com/v1"
    "api_key": str,   # 密钥（env 变量引用则尝试展开）
    "model": str,     # 模型名，如 "qwen-max"
    "source": str,    # 来源文件路径
    "enabled": bool   # 默认全部 True，除非文件明确标注 false
}
```

#### 第 3 步：展示模型列表，让用户勾选

将解析出的模型列表以交互方式展示给用户：

```
🔍 找到以下已配置的模型，请勾选你想对比的几个：
（直接回复对应编号，可多选，空格或逗号分隔，默认全选）

[A] qwen-max   → https://dashscope.aliyuncs.com/v1   from ~/.qclaw/gateway.json
[B] gpt-4o     → https://api.openai.com/v1           from ~/.env
[C] llama-3.1  → http://localhost:11434/v1           from ~/.ollama/
[D] hunyuan    → https://api.hunyuan.cloud.tencent.com/v1  from ~/.qclaw/config.json

请选择（如 A C）：__
```

**用户回复处理**：
- 用户回复编号 → 映射到对应模型
- 用户回复 `全部` 或直接回车 → 全选
- 用户回复 `取消` → 终止本次 Skill 执行
- 用户回复无效 → 提示重新输入，最多重复 2 次

#### 第 4 步：并行调用选中的模型

对每个选中的模型，**并行**发起 OpenAI 兼容 API 调用：

```python
import openai
import asyncio

async def ask_model(model_config, question):
    client = openai.OpenAI(
        base_url=model_config["base_url"],
        api_key=model_config["api_key"]
    )
    response = client.chat.completions.create(
        model=model_config["model"],
        messages=[{"role": "user", "content": question}]
    )
    return model_config["id"], response.choices[0].message.content

# 并行
tasks = [ask_model(cfg, question) for cfg in selected_models]
results = await asyncio.gather(*results)
```

**注意**：
- 必须在**同一条消息内**发出多个 Agent 工具调用，实现真并发
- 单个模型调用失败 → 该栏显示"❌ 调用失败：{简要原因}"，不影响其他模型
- 全部失败 → 提示用户检查配置，Skill 终止

#### 第 5 步：格式化对比输出

将结果以多栏对比格式呈现，栏数等于选中模型数（通用，不限 3 栏）：

```
╔════════════════════════════════════════════════════════════════╗
║                    📊 问题：{用户问题}                    ║
╠══════════════════╦══════════════════╦══════════════════════╣
║ [A] qwen-max     ║ [B] gpt-4o      ║ [C] llama-3.1        ║
╠══════════════════╬══════════════════╬══════════════════════╣
║ {回答内容A}       ║ {回答内容B}      ║ {回答内容C}          ║
╚══════════════════╩══════════════════╩══════════════════════╝
```

**格式规则**：
- 回答内容超长时，每栏独立截断（保留约 800 字），末尾加"…"
- 调用失败的模型栏显示错误原因
- 最后加一行注："以上回答由各模型独立生成，仅供参考"

---

## 3. 错误处理

| 场景 | 处理方式 |
|------|---------|
| LLM 返回 `NONE` 或无路径 | 提示"未找到任何配置文件，请确认是否已配置大模型"，给出建议路径 |
| 所有配置文件均不可读 | 提示用户手动提供 base_url + model + key |
| 单模型调用失败 | 该栏显示错误，其余正常显示 |
| 全部调用失败 | 汇总原因，提示检查配置 |
| 用户未勾选任何模型 | 提示后重新展示列表，最多重复 2 次 |
| 网络超时 | 该栏显示超时，其余正常 |
| API Key 无效 | 该栏显示认证失败，提示用户检查 Key |

---

## 4. 安全约束

- **不读取 Skill 目录外的未经用户确认的路径**
- **API Key 不出现在回复内容中**，用 `***` 脱敏后显示
- **配置文件仅读不写**，不做任何修改
- **不将 Key 写入任何日志或 Skill 文件**

---

## 5. 快速命令参考

```python
# 并行调用多个模型
import asyncio
import openai

async def ask_model(model_config, question):
    client = openai.OpenAI(
        base_url=model_config["base_url"],
        api_key=model_config["api_key"]
    )
    response = client.chat.completions.create(
        model=model_config["model"],
        messages=[{"role": "user", "content": question}]
    )
    return model_config["id"], response.choices[0].message.content

# 并行调用
tasks = [ask_model(cfg, question) for cfg in selected_models]
results = await asyncio.gather(*tasks)
```

---

*最后更新：2026-08-11*
