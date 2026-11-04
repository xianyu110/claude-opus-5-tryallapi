# Claude Opus 5 API 调用教程：用 TryAllAPI 做 OpenAI 兼容对话接入

> **Claude Opus 5**（TryAllAPI 模型 ID：`claude-opus-5`）Claude Opus 5 是 Claude Max 的默认模型，也是 Claude Pro 最强大的模型，它是一个深思熟虑、积极主动、具有成本效益的代理模型，提供几乎与旗舰 Claude Fable 5 相媲美的前沿智能，成本仅为其一半，并为编码和专业知识工作设定了新的行业标准。价格接口标签为「对话、工具、识图」。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场 / 价格页选型、读基价与分组倍率，到 `POST /v1/chat/completions` 的完整接入流程。价格与分组倍率取数 **2026-11-04（北京）**，**以页面实时为准**；示例仅为请求形态说明，未跑通真实长任务计费。

**Keywords:** Claude Opus 5 · claude-opus-5 · TryAllAPI · OpenAI 兼容 · `/v1/chat/completions` · Anthropic

---

## 目录

- [一、为什么用聚合站调 Claude](#一为什么用聚合站调-Claude)
- [二、在价格页 / 模型广场找到 Claude Opus 5](#二在价格页--模型广场找到-Claude-Opus-5)
- [三、模型怎么选：与相邻档对照](#三模型怎么选与相邻档对照)
- [四、价格怎么读](#四价格怎么读)
- [五、调用流程：Key → Base URL → chat/completions](#五调用流程key--base-url--chatcompletions)
- [六、curl / SDK 最小示例](#六curl--sdk-最小示例)
- [七、常见注意](#七常见注意)
- [FAQ](#faq)
- [八、小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调 Claude

只用官方控制台、能走官方充值开票时，直连往往最干净；如果项目里还要同时调 GPT、Claude、Gemini、国内模型，为每家维护一套 Key 和重试逻辑也比较散。

聚合站（如 [TryAllAPI](https://tryallapi.com/)）的常见价值：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可试不同模型；
2. **OpenAI 兼容形态**：Base URL 固定为 `https://tryallapi.com/v1`，Cursor / Dify / 各类 SDK 改地址即可；
3. **模型广场可对照**：标签、分组倍率、端点类型一目了然。

代价是多一跳：延迟与可用性取决于分组上游，建议先用小流量探针。注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在价格页 / 模型广场找到 Claude Opus 5

打开 [价格页](https://tryallapi.com/pricing) 或模型广场，按供应商 **Anthropic**、类型 **对话**、关键词 `claude-opus-5` 筛选。详情卡截图如下：

![Claude Opus 5 模型广场详情卡：模型 ID、API 与分组价格](model-square.png)

详情卡要点（取自价格接口 2026-11-04（北京） 快照）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `claude-opus-5` |
| **供应商** | Anthropic |
| **标签** | 对话、工具、识图 |
| **端点类型** | `anthropic`、`openai`（对话主路径：`POST /v1/chat/completions`） |
| **可用分组** | AWS-Bedrock-1、AWS-Bedrock-2、AWS-Claude-1、AWS-Claude-2、AWS-Claude-3、Anthropic-Claude-1、Azure-Claude-1、Claude-Code-1、Claude-Code-2、Claude-Code-3、Kiro-Claude-1 |
| **计费** | 按量（token） |

文中 UI 品牌可能显示为模型广场 / 第三方壳，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与价格页为准。

---

## 三、模型怎么选：与相邻档对照

只根据价格接口可见信息做对照，**不编造未出现的能力差异**。基价按价格接口换算（$/1M，未乘分组倍率）：

| 模型 ID（可见） | 基价 输入 / 输出 / 缓存读 | 怎么理解 |
| --- | --- | --- |
| **`claude-opus-5`** | $5 / $25 / $0.5 | 本文主线：Claude Opus 5；见上文描述 |
| `claude-opus-5-5` | $4 / $20 / $0.2 | 同系列 Opus 5.5；基价不同对照 |
| `claude-opus-4-8` | $5 / $25 / $0.5 | 相邻 Opus 4.8；同基价可见对照 |
| `claude-fable-5-1` | $10 / $50 / $0.25 | Fable 5.1 旗舰档；更高基价对照 |
| `claude-sonnet-5` | $2 / $10 / $0.2 | Sonnet 5 中档；更低基价对照 |

实务上：经 TryAllAPI 调用统一写精确字符串 `claude-opus-5`；遇限流或分组无货，再在同厂商可见 ID 之间切换，而不是改 Base URL。

---

## 四、价格怎么读

对话模型常见公式：**≈ 基价 × 分组倍率**（以控制台为准）。取数快照：**2026-11-04（北京）**；**以页面实时为准**。

换算约定：基价输入 $= model_ratio×2；输出 $= 输入×completion_ratio；缓存读 $= 输入×cache_ratio；缓存创建（若有）= 输入×cache_creation_*_ratio。分组估价 = 基价 × group_ratio。

### 基价（每百万 tokens，由价格接口换算）

| 计费项 | 基价（$/1M，未乘分组） |
| --- | --- |
| 输入 | **$5** |
| 输出 | **$25** |
| 缓存读 | **$0.5** |

公开接口按 **基价 × group_ratio** 估算即可；**实际扣费以控制台为准**。

本模型快照未见缓存创建倍率字段。接口描述提到 Agent / 编码相关场景；具体能力边界以价格页与控制台为准。

### 分组价格估算（基价 × group_ratio，$/1M）

| 分组 | group_ratio | 输入估 | 缓存读估 | 输出估 |
| --- | --- | --- | --- | --- |
| **Kiro-Claude-1** | 0.194128 | ≈ $0.9706 | ≈ $0.0971 | ≈ $4.8532 |
| **Claude-Code-1** | 0.388234 | ≈ $1.9412 | ≈ $0.1941 | ≈ $9.7058 |
| **Claude-Code-2** | 0.647064 | ≈ $3.2353 | ≈ $0.3235 | ≈ $16.1766 |
| **AWS-Bedrock-1** | 0.970596 | ≈ $4.853 | ≈ $0.4853 | ≈ $24.2649 |
| **AWS-Claude-1** | 0.970596 | ≈ $4.853 | ≈ $0.4853 | ≈ $24.2649 |
| **Azure-Claude-1** | 0.970596 | ≈ $4.853 | ≈ $0.4853 | ≈ $24.2649 |
| **Claude-Code-3** | 0.99 | ≈ $4.95 | ≈ $0.495 | ≈ $24.75 |
| **AWS-Bedrock-2** | 1.294128 | ≈ $6.4706 | ≈ $0.6471 | ≈ $32.3532 |
| **AWS-Claude-2** | 1.941192 | ≈ $9.706 | ≈ $0.9706 | ≈ $48.5298 |
| **AWS-Claude-3** | 2.42 | ≈ $12.1 | ≈ $1.21 | ≈ $60.5 |
| **Anthropic-Claude-1** | 2.42 | ≈ $12.1 | ≈ $1.21 | ≈ $60.5 |

**最终扣费以控制台流水为准**。卡片展示价取自最低 `group_ratio` 分组 **Kiro-Claude-1**（与定价页公开数字一致）。

---

## 五、调用流程：Key → Base URL → chat/completions

```text
控制台创建 API Key（选本模型 enable_groups 中的分组）
  → Base URL = https://tryallapi.com/v1
  → POST /v1/chat/completions   （model=claude-opus-5）
  → 读 choices[0].message.content（或流式 delta）
```

建议：

- **超时**：长输出 / 思考模式建议开流式并调高 timeout；
- **密钥**：只放环境变量（如 `TRYALLAPI_KEY`），勿提交仓库；
- **端点类型**：价格接口标注端点类型：`anthropic`、`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

---

## 六、curl / SDK 最小示例

占位 Key `sk-xxx`（**不要**填入真实 Key）。

### 1）OpenAI 兼容 chat/completions

```bash
curl -sS https://tryallapi.com/v1/chat/completions \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "claude-opus-5",
    "messages": [
      {"role": "user", "content": "把这份多步调研任务拆成可执行的验收清单（不超过 8 条）"}
    ]
  }'
```

### 2）Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",  # 建议改为读取环境变量
    base_url="https://tryallapi.com/v1",
    timeout=180,
)

r = client.chat.completions.create(
    model="claude-opus-5",
    messages=[{"role": "user", "content": "用三句话说明如何给这段多步 agent 工作流写回归用例。"}],
)
print(r.choices[0].message.content)
```

Cursor：Settings → Models → OpenAI API Key + Override Base URL `https://tryallapi.com/v1` → 自定义模型 `claude-opus-5`。Dify：OpenAI-API-compatible，endpoint 填 `https://tryallapi.com/v1`。

---

## 七、常见注意

1. **模型名**：请求体 `"model"` 必须写广场上的精确 ID `claude-opus-5`（含连字符 / 点号，大小写敏感）；写混成简称或漏掉后缀容易 404。
2. **基价 × 倍率**：估算用价格接口快照；峰谷 / 促销字段若页面另有说明，以控制台为准。
3. **分组**：403 多为分组无货，换 `enable_groups` 里其他分组；低倍率分组先抽检质量与延迟。
4. **报错速查**：401 查 Bearer / 令牌；404 确认模型名且 base_url 含 `/v1`；429 退避或换分组。
5. **合规**：第三方聚合不是 Anthropic 官方服务，敏感代码先看条款。

---

## FAQ

### Claude Opus 5 的模型 ID 是什么？

TryAllAPI 上是 **`claude-opus-5`**。请求体 `"model"` 填该字符串。

### TryAllAPI 的对话 endpoint 是什么？

- Base URL：`https://tryallapi.com/v1`
- 对话：`POST /v1/chat/completions`
- 价格接口标注端点类型：`anthropic`、`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

### 基价大概是多少？

按 2026-11-04（北京） 价格接口快照：输入约 $5 / 输出约 $25 / 缓存读约 $0.5（$/1M，未乘分组）。再 × `group_ratio` 得分组估价；**以页面实时为准**。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 Claude Opus 5 接到业务里，核心是：**在价格页确认 model id 与 OpenAI 兼容 Base URL → 读懂基价 × 分组倍率 → 用 chat/completions 先跑通非流式 ping，再开流式 / 接工具调用**。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [tryallapi.com/pricing](https://tryallapi.com/pricing) |

价格取数 2026-11-04（北京），以页面实时为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非 Anthropic 官方产品。
