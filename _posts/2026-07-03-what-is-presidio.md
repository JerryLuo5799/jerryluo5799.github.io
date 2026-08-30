---
layout: post
title:  "Presidio 入门：微软开源的 PII 脱敏工具，刚刚交给了社区"
date:   2026-07-03 09:30:00 +0800--
categories: [AI]
tags: [Presidio]
---

## 前言

先问一个很多团队都答不上来的问题：**你们的日志里，有没有用户的手机号？**

大概率是有的。异常堆栈打印了整个请求体、排查问题时随手 `LogInformation(user.ToString())`、导出一份"脱敏过的"数据给外包做测试——这些地方都是 PII（个人身份信息，Personally Identifiable Information）泄漏的重灾区。

真要动手治理的时候，你会发现事情比想象中麻烦：手机号还好，正则能搞定；但**人名怎么办？地址怎么办？**"张伟"和"张伟路 32 号"，一个是人名一个是地址，正则区分不了。

这就是 [Presidio](https://github.com/data-privacy-stack/presidio?wt.mc_id=MVP_324329) 要解决的问题。它是一套开源的 PII 识别与脱敏工具，10.6k star，MIT 协议。而且它最近发生了一件挺重要的事——**它换东家了**。

下面先说这个变化，再讲它是什么、怎么跑起来。

## 一、先说这个重要变化：它从微软搬到社区了

如果你以前用过 Presidio，或者搜到的资料还指向 `microsoft/presidio`，**请注意仓库已经迁移了**：

| | 迁移前 | 迁移后 |
| --- | --- | --- |
| 仓库 | `github.com/microsoft/presidio` | `github.com/data-privacy-stack/presidio` |
| 归属 | 微软自有项目 | 独立的社区治理项目 |
| 容器镜像 | `mcr.microsoft.com/presidio-*` | `ghcr.io/data-privacy-stack/presidio-*` |
| 协议 | MIT | MIT（不变） |

官方的迁移公告在 2026 年 6 月底发布，说明了几件事：

* **协议不变，仍然是 MIT**，功能、API、文档都保持可用。
* **微软支持这次转型**，项目转为社区志愿者维护，不再由某个商业实体拥有和运营。
* **技术指导委员会（TSC）会扩充**，纳入微软之外的维护者。
* 对现有用户——**包括在 Azure 上使用 Presidio 的组织——使用体验不变**，现有集成和依赖预期继续正常工作。

### 有一件事你必须动手改

公告里最实际的一条是**容器镜像地址变了**：

> 旧的 `mcr.microsoft.com/presidio-*` 镜像对老 tag 仍然可以拉取，但**不再更新**。

也就是说，如果你的 `docker-compose.yml` 或 K8s 编排里还写着 `mcr.microsoft.com/presidio-analyzer:latest`，它**不会报错，只会悄悄停留在旧版本上**——这种问题最难发现。现在就改掉：

```yaml
# 旧的（不再更新）
image: mcr.microsoft.com/presidio-analyzer:latest

# 新的
image: ghcr.io/data-privacy-stack/presidio-analyzer:latest
```

顺便提醒：生产环境**别用 `latest`**，去 GitHub Packages 页面挑一个明确的版本号钉住。

## 二、Presidio 是什么

名字来自拉丁语 *praesidium*，意思是"保护、守卫"。

一句话概括：**它帮你在文本、图片和结构化数据里找出敏感信息，然后按你的要求处理掉。**

它不是一个大而全的服务，而是四个各司其职的 Python 包：

![Presidio 的四个模块](/assets/imgs/presidio-modules.svg)

绝大多数人只需要左边两个：**analyzer 负责"找"，anonymizer 负责"改"**。这两步是分开的，这个设计很关键——因为"找出来了"和"要怎么处理"是两个独立的决策，同一批识别结果，你可以选择替换、遮蔽、加密或者干脆删掉。

## 三、五分钟跑起来

### 方式一：Python 包

```bash
pip install presidio-analyzer
pip install presidio-anonymizer
python -m spacy download en_core_web_lg
```

注意第三行：Presidio 默认用 spaCy 做自然语言处理，**这个模型得单独下载**，大约 500MB。第一次跑忘了下载是最常见的报错。

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

text = "My phone number is 212-555-5555"

# 初始化，会加载 spaCy 模型和全部内置识别器（第一次比较慢）
analyzer = AnalyzerEngine()

results = analyzer.analyze(text=text, entities=["PHONE_NUMBER"], language="en")
print(results)
# [type: PHONE_NUMBER, start: 19, end: 31, score: 0.75]

# 把识别结果交给 anonymizer 处理
anonymizer = AnonymizerEngine()
print(anonymizer.anonymize(text=text, analyzer_results=results).text)
# My phone number is <PHONE_NUMBER>
```

留意 `analyze()` 的返回值：**它给的不是脱敏后的文本，而是一份"哪个位置有什么"的清单**——实体类型、起止位置、置信度分数。这份清单原样传给 `anonymize()`，才得到最终结果。

### 方式二：Docker（推荐给非 Python 团队）

如果你的技术栈不是 Python，别折腾 pip，直接起容器当 HTTP 服务用：

```bash
docker pull ghcr.io/data-privacy-stack/presidio-analyzer
docker pull ghcr.io/data-privacy-stack/presidio-anonymizer

# 容器内默认监听 3000 端口，这里映射到宿主机的 5002 / 5001
docker run -d -p 5002:3000 ghcr.io/data-privacy-stack/presidio-analyzer:latest
docker run -d -p 5001:3000 ghcr.io/data-privacy-stack/presidio-anonymizer:latest
```

起来之后 `curl` 一下：

```bash
curl -X POST http://localhost:5002/analyze \
  -H "Content-Type: application/json" \
  -d '{"text":"My phone number is 212-555-5555","language":"en"}'
```

analyzer 提供的接口很少，好记：

| 接口 | 作用 |
| --- | --- |
| `POST /analyze` | 识别 PII，返回实体清单 |
| `GET /recognizers` | 当前加载了哪些识别器 |
| `GET /supportedentities` | 支持哪些实体类型 |
| `GET /health` | 健康检查 |

anonymizer 对应的是 `POST /anonymize`、`POST /deanonymize`（还原）和 `GET /anonymizers`。

怎么从 .NET 调这套接口，[另一篇文章](/2026/07/10/call-presidio-from-dotnet/)里有完整的例子，这里先不展开。

## 四、它是怎么"认出"PII 的

这是理解 Presidio 最关键的一节。很多人以为它就是个大正则库，其实不是。

![识别与脱敏的流程](/assets/imgs/presidio-analyze-flow.svg)

### 四类识别器同时工作

Presidio 里每一种 PII 都由一个 **Recognizer（识别器）** 负责，识别器分四类：

| 类型 | 适合什么 | 特点 |
| --- | --- | --- |
| **正则表达式** | 手机号、邮箱、IP 地址 | 快，但容易误报 |
| **校验位算法** | 银行卡（Luhn）、各国身份证 | 能算出真假，最准 |
| **词表（Deny-list）** | 固定词：职务、科室名称 | 简单粗暴，完全可控 |
| **NER 模型** | 人名、地名、机构名 | 写不出规则的靠它 |

**最后一类是 Presidio 真正的价值所在。** "张伟"为什么是人名？没有规则可写，只能靠训练过的模型判断。这也是为什么它要依赖 spaCy——那 500MB 就是干这个的。

### 每个结果都带一个分数

识别出来的每个实体都有一个 0 到 1 的置信度：

```
PHONE_NUMBER  [6,17]   0.85
EMAIL_ADDRESS [21,40]  1.00    ← 邮箱格式太明确了，满分
DATE_TIME     [50,58]  0.60    ← 不太确定
```

你可以用 `score_threshold` 卡掉低分结果：

```python
results = analyzer.analyze(text=text, language="en", score_threshold=0.6)
```

**这个参数是准确率和召回率之间的旋钮**：调高，漏检增多；调低，误报增多。没有普适的最优值，得拿你自己的真实数据试。

## 五、找到之后怎么处理

`anonymize()` 的第三个参数 `operators` 决定处理方式。Presidio 内置了这些：

| 处理方式 | 效果 | 典型场景 |
| --- | --- | --- |
| `replace` | 换成 `<PHONE_NUMBER>` 之类的占位符 | 默认行为，给人看的日志 |
| `redact` | 直接删掉 | 不需要知道这里原本有东西 |
| `mask` | 部分遮蔽，如 `138****0000` | 客服工单，要留一点辨识度 |
| `hash` | 换成哈希值 | 需要"同一个人"可对比，但不需要还原 |
| `encrypt` | 加密，**可以还原** | 送给第三方处理，之后要拿回原文 |
| `keep` | 保留不动 | 白名单，某些实体故意不处理 |
| `custom` | 你自己的函数 | 比如用假数据替换 |

不同实体用不同策略：

```python
from presidio_anonymizer.entities import OperatorConfig

result = anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators={
        # 手机号只遮蔽中间，末尾保留
        "PHONE_NUMBER": OperatorConfig("mask", {
            "masking_char": "*",
            "chars_to_mask": 4,
            "from_end": False,
        }),
        # 人名直接换成固定字符串
        "PERSON": OperatorConfig("replace", {"new_value": "<某用户>"}),
        # 其他没单独指定的，一律删掉
        "DEFAULT": OperatorConfig("redact"),
    },
)
print(result.text)
```

`hash` 那一行值得单独说：**它让"同一个手机号"在脱敏后仍然是同一个值**。做用户行为分析时，你需要知道"这两条日志是同一个人"，但不需要知道他是谁——`hash` 正好卡在这个点上。

## 六、几个必须知道的限制

### 1. 它不保证找全

这是官方 README 里用警告框标出来的原话，我原样转述：

> Presidio 可以帮助识别非结构化/结构化文本中的敏感数据。但由于使用的是自动检测机制，**无法保证 Presidio 能找到所有敏感信息**。因此应当同时采用其他系统和防护措施。

请把这句话当真。**Presidio 是纵深防御里的一层，不是唯一一层。** 拿它当"合规完成"的证明，迟早出事。

### 2. 默认只认英文

内置配置里的模型和识别器都是针对英文的。中文文本丢进去，`en_core_web_lg` 连词都切不对，人名基本认不出来。

### 3. 支持 20 个国家，其中没有中国

我数了一下官方的实体清单：美国、英国、西班牙、意大利、波兰、新加坡、澳大利亚、印度、芬兰、韩国、尼日利亚、菲律宾、加拿大、瑞典、南非、泰国、土耳其、德国……以及一组通用实体（邮箱、信用卡、IP、加密货币钱包等）和医疗类实体。

**内置识别器里没有任何中国相关的条目**——身份证、手机号、统一社会信用代码，一个都没有。

好消息是这套东西天生就是为扩展设计的，自己加不难——[让 Presidio 认识中文](/2026/07/17/presidio-chinese-pii/)里有完整的做法。

### 4. 容器冷启动慢

analyzer 容器启动时要加载 NLP 模型，**第一次请求可能要等十几秒**。健康检查的 `start_period` 要留够，否则编排系统会以为它挂了然后反复重启。

## 总结

* **Presidio 刚从 `microsoft/presidio` 迁到 `data-privacy-stack/presidio`**，MIT 协议不变，但**镜像地址必须从 `mcr.microsoft.com` 改到 `ghcr.io`**，旧地址不再更新。
* 四个模块，文本场景只需 **analyzer（找）+ anonymizer（改）** 这一对。
* 识别靠四类识别器协同：**正则、校验位、词表、NER 模型**，每个结果带置信度分数，用 `score_threshold` 调节松紧。
* 处理方式有 7 种，其中 `hash` 能保持可对比性、`encrypt` 能还原，这两个最容易被忽略。
* **它不保证找全，默认只认英文，内置识别器里没有中国。**

其中几个问题另有专文：

* **[从 .NET 调用 Presidio](/2026/07/10/call-presidio-from-dotnet/)**——它是 Python 项目，仓库里连一个 `.cs` 文件都没有，这条路得自己铺。
* **[让它认识中国的身份证和手机号](/2026/07/17/presidio-chinese-pii/)**——正则、校验位和中文 NER 模型的组合拳。
* **[给大模型应用做脱敏闸门](/2026/07/24/presidio-llm-pii-gateway/)**——让客户信息不出机房。

## 延伸阅读

* [Presidio 仓库（新地址）](https://github.com/data-privacy-stack/presidio?wt.mc_id=MVP_324329)
* [官方文档](https://data-privacy-stack.github.io/presidio?wt.mc_id=MVP_324329)
* [在线 Demo（Hugging Face）](https://huggingface.co/spaces/presidio/presidio_demo)
