---
layout: post
title:  "给大模型加一道脱敏闸门：用 Presidio 让客户信息不出机房"
date:   2026-07-24 09:30:00 +0800--
categories: [AI]
tags: [Presidio]
---

## 前言

这是 Presidio 系列的最后一篇。前三篇讲了[它是什么](/2026/07/03/what-is-presidio/)、[怎么从 .NET 调用](/2026/07/10/call-presidio-from-dotnet/)、[怎么让它认识中文](/2026/07/17/presidio-chinese-pii/)。这一篇讲它现在最热的一个用法。

先说场景。你们上线了一个 AI 客服助手，客服人员这样用它：

> 「客户张伟（手机 13800138000）投诉订单迟迟未发货，帮我起草一个回复。」

这句话会原封不动地发到 OpenAI 或者别的云端模型。**真实姓名和手机号，就这样离开了你的机房。**

这不是假想的风险。一旦启用了 AI 助手，员工往对话框里粘贴的东西会远超你的预期——完整的工单、数据库查询结果、甚至整段客户资料。而这些内容送到哪去了、被留存多久、会不会进训练集，很多时候不在你的控制范围内。

Presidio 能在中间加一道闸门。

## 一、整体思路

![LLM 脱敏闸门](/assets/imgs/presidio-llm-gateway.svg)

核心就一句话：**在数据离开你的边界之前脱敏，在结果回来之后还原。**

云端模型自始至终看到的都是占位符，它照样能完成「起草一封道歉回复」这个任务——因为**写回复这件事根本不需要知道客户真名**。

## 二、先做一个决定：需不需要还原

这是设计上的第一个岔路口，选错了后面全是麻烦。

| | **不可逆** | **可逆** |
| --- | --- | --- |
| 用什么 | `replace` / `redact` / `hash` | `encrypt` / 映射表 |
| 结果 | 原文永远拿不回来 | 能还原成原文 |
| 适合 | 数据分析、日志、内部问答 | 要把结果给客户看的场景 |
| 风险 | 低——密文都没有，泄不了 | 高——密钥或映射表一旦泄漏就全完了 |

**判断标准很简单：模型的输出会不会直接送到最终用户面前？**

* 「帮我总结这批工单的共同问题」→ **不可逆**。总结里根本不该出现具体某个人。
* 「帮我起草回复这位客户」→ **可逆**。回复要发出去，抬头得是真名。

**默认选不可逆。** 只有确实需要还原时才用可逆方案——因为可逆意味着你要开始管密钥了，那是另一套麻烦。

## 三、可逆方案：用 encrypt / decrypt

Presidio 内置了 AES 加密的 operator，加密和解密用同一个密钥：

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine, DeanonymizeEngine
from presidio_anonymizer.entities import OperatorConfig

# 密钥长度必须是 128 / 192 / 256 位，也就是 16 / 24 / 32 个字符
crypto_key = "WmZq4t7w!z%C&F)J"

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

text = "Please draft a reply to James Bond about his delayed order."

# ---- 出境前：识别 + 加密 ----
findings = analyzer.analyze(text=text, language="en")
masked = anonymizer.anonymize(
    text=text,
    analyzer_results=findings,
    operators={"DEFAULT": OperatorConfig("encrypt", {"key": crypto_key})},
)

print(masked.text)
# Please draft a reply to <一串密文> about his delayed order.

# 这一步很关键：记下每个被加密实体的位置，还原时要用
encrypted_entities = masked.items

# ---- 把 masked.text 发给大模型，拿到回复 ----
llm_reply = call_your_llm(masked.text)

# ---- 入境后：还原 ----
deanonymizer = DeanonymizeEngine()
final = deanonymizer.deanonymize(
    text=llm_reply,
    entities=encrypted_entities,
    operators={"DEFAULT": OperatorConfig("decrypt", {"key": crypto_key})},
)
print(final.text)
```

两个必须注意的点：

**1. 密钥长度有硬性要求。** 必须是 16、24 或 32 个字符，对应 AES-128/192/256。长度不对会直接抛异常。而且这个密钥**绝对不能硬编码进代码**——放 Azure Key Vault 或者你们用的任何密钥管理服务里。

**2. `items` 一定要留着。** `anonymize()` 返回的不只是文本，还有一份「哪个位置是什么实体」的清单。还原的时候必须把它传进去——**它不是可选的调试信息，是还原的必要输入**。

## 四、现成方案：LiteLLM 代理

如果你不想自己写这一层，[LiteLLM](https://github.com/BerriAI/litellm) 已经把 Presidio 做成了内置回调。它是一个 LLM 网关，你的应用调它，它再去调真正的模型。

```bash
export PRESIDIO_ANALYZER_API_BASE="http://localhost:5002"
export PRESIDIO_ANONYMIZER_API_BASE="http://localhost:5001"
```

```yaml
# config.yaml
model_list:
  - model_name: my-openai-model
    litellm_params:
      model: gpt-4o

litellm_settings:
  callbacks: ["presidio"]
  output_parse_pii: true    # 让它自动把返回结果里的占位符换回原值
```

```bash
litellm --config /path/to/config.yaml
```

`output_parse_pii: true` 这一行做的事情正是我们上面手写的那套：

1. 用户输入：`hello world, my name is Jane Doe. My number is: 034453334`
2. 送给模型的：`hello world, my name is [PERSON]. My number is: [PHONE_NUMBER]`
3. 模型返回的：`Hey [PERSON], nice to meet you!`
4. 还给用户的：`Hey Jane Doe, nice to meet you!`

**这个方案最大的好处是对应用无侵入**——你的代码还是按 OpenAI 的接口调，只是把 base URL 指向 LiteLLM。想给多个应用统一加脱敏，这是最省事的路子。

它还支持在请求里带 **ad-hoc 识别器**，也就是上一篇提到的那个机制：不用改镜像，直接在配置里挂一个 JSON 文件补充自定义规则。

## 五、真正的难点：脱敏会让模型变笨

这一节是这篇文章里最重要的部分，也是各种教程最少提到的部分。

**把 PII 换成占位符，是在删除信息。而信息删多了，模型就答不好了。**

### 问题 1：占位符会让模型分不清人

```
原文：张伟向李娜转账 5000 元，李娜确认收到。
脱敏：<PERSON> 向 <PERSON> 转账 5000 元，<PERSON> 确认收到。
```

模型现在完全无法判断谁转给谁、谁确认了——**三个不同的人被压成了同一个符号**。

**解决办法：给占位符编号**，让不同的人对应不同的记号：

```
<PERSON_1> 向 <PERSON_2> 转账 5000 元，<PERSON_2> 确认收到。
```

Presidio 的内置 operator 不会自动做这件事，需要你在中间那一步自己处理（还记得上一篇说的「识别结果在中间是可以被你干预的」吗？就是用在这种地方）——按原文值去重，给每个不同的值分配一个编号，同时维护一张编号到原值的映射表用于还原。

### 问题 2：占位符会破坏语气

```
脱敏后：尊敬的 <PERSON> 您好……
```

模型不知道这是个人名还是公司名，写出来的回复会很别扭。**更好的做法是用假名替换**——把「张伟」换成「王强」而不是 `<PERSON>`。

模型看到的是一个正常的中文姓名，行文自然流畅；而真实姓名依然没有出境。还原时按映射表换回来即可。Presidio 支持通过 `custom` operator 接入任意替换逻辑，配合 Faker 这类假数据库就能做到。

**代价是你要维护一张映射表**，而映射表本身就是敏感数据——它需要和加密密钥同等级别的保护。

### 问题 3：流式输出很难还原

如果你用的是流式响应（打字机效果），**占位符可能被切成两半分批送达**：`<PER` 一个包，`SON_1>` 下一个包。逐块做字符串替换会失败。

处理办法是在还原层维护一个缓冲区，遇到疑似占位符开头就先攒着，凑完整了再替换。**这部分工程量比想象中大**，如果你的场景对首字延迟不敏感，直接关掉流式会省很多事。

### 问题 4：RAG 场景要想清楚在哪脱敏

检索增强（RAG）里有两个时机可以脱敏，选择完全不同：

* **入库时脱敏**：向量库里存的就是脱敏后的内容。**最安全**，但检索质量会下降——用户搜「张伟的订单」，库里根本没有「张伟」。
* **检索后脱敏**：库里存原文，取出来送给模型之前才脱敏。**检索质量不受影响**，但向量库本身成了敏感数据存储，得按敏感级别来保护。

**没有标准答案，取决于你的向量库归谁管、在哪部署。** 但这个决定必须在设计阶段就做，事后改动成本极高。

## 六、几个容易被忽略的点

### 1. 别忘了它不保证找全

这句话我在这个系列里说了四遍，因为它在这个场景下后果最严重。

Presidio 漏掉一个手机号，意味着**这个手机号真的被发到云端去了**。所以：

* **不要把 Presidio 当作唯一的防线。** 该有的访问控制、数据分级、员工培训一样都不能少。
* **要做审计。** 把每次识别的命中记录下来，定期抽查有没有明显该识别却没识别的模式。
* **要限制入口。** 与其指望脱敏兜住一切，不如从产品设计上就不让员工能随手粘贴整段客户资料。

### 2. 成本和延迟

每次调用多了两次 HTTP 往返（analyze + anonymize）。相比大模型本身几百毫秒到几秒的耗时，这点开销通常可以忽略——**但前提是 Presidio 服务和你的应用在同一个网络里**。跨区域部署会让这个开销变得很显眼。

如果只关心少数几类实体，记得用 `entities` 参数缩小范围，这在高频调用场景下能省不少。

### 3. 什么时候不需要这一层

**如果模型是私有部署、跑在你自己的机房里，这一层就没有存在的必要。** 数据根本没有离开边界，脱敏只会白白降低模型效果。

同样，如果你用的是有明确合规承诺、且部署在你所在区域的服务（比如某些企业版云服务的数据不出境条款），**先看清楚合同**——有可能你需要的不是技术方案，而是一份已经签好的协议。

**别为了上技术而上技术。** 这一层的价值完全建立在「数据确实要出境」这个前提上。

## 总结

* 核心思路：**出境前脱敏，回来后还原**，云端模型看到的永远是占位符。
* 第一个决定是**要不要还原**：输出直接给最终用户看的才需要可逆方案，其余一律用不可逆的，默认选不可逆。
* 可逆方案用 `encrypt` / `decrypt`：**密钥必须是 16/24/32 字符且不能硬编码**，`anonymize()` 返回的 `items` 是还原的必要输入。
* 不想自己写，用 **LiteLLM 代理**：配置里加一行 `callbacks: ["presidio"]`，对应用无侵入。
* **最大的难点不是脱敏，是脱敏之后模型会变笨**：占位符要编号区分不同的人，用假名替换比用占位符更保语气，流式输出的还原要额外处理，RAG 要提前想清楚在哪一步脱敏。
* **它依然不保证找全**——这一层是纵深防御的一环，不是免责声明。

至此，Presidio 这个系列的四篇就写完了：[入门与项目迁移](/2026/07/03/what-is-presidio/)、[.NET 集成](/2026/07/10/call-presidio-from-dotnet/)、[中文识别](/2026/07/17/presidio-chinese-pii/)，以及这一篇。

## 延伸阅读

* [LiteLLM 的 Presidio 集成文档](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2)
* [Presidio 加解密教程](https://data-privacy-stack.github.io/presidio/tutorial/12_encryption/)
* [让 Presidio 认识中文（本系列上篇）](/2026/07/17/presidio-chinese-pii/)
