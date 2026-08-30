---
layout: post
title:  "让 Presidio 认识中文：身份证、手机号与自定义识别器实战"
date:   2026-07-17 09:30:00 +0800--
categories: [Tools]
tags: [Presidio]
---

## 前言

前两篇（[入门](/2026/07/03/what-is-presidio/)、[.NET 集成](/2026/07/10/call-presidio-from-dotnet/)）里，我一直在用英文示例。这一篇要面对真正的问题：

**Presidio 默认配置对中文基本是无效的。**

先看一眼有多无效：

```python
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()   # 默认配置，en_core_web_lg
text = "张伟的手机是 13800138000，身份证 11010519491231002X"

print(analyzer.analyze(text=text, language="en"))
```

结果大概是这样：人名「张伟」认不出来，身份证认不出来，手机号运气好的话会被当成某种数字实体，但类型多半是错的。

原因不复杂：

1. **NLP 模型是英文的**。`en_core_web_lg` 是在英文语料上训练的，它连中文的词都切不对，更别说判断哪个词是人名。
2. **内置识别器没有中国**。我数过官方的实体清单，覆盖了美国、英国、西班牙、印度、韩国、泰国等 20 个国家，**里面没有中国**。身份证、统一社会信用代码，一个都没有。
3. **上下文提示词是英文的**。识别器会看目标附近有没有 `phone`、`mobile` 这类词来加分，中文里这些词是「手机」「电话」，对不上。

这三条要分别解决。好消息是 Presidio 从设计上就是给人扩展的，动手不难。

## 一、第一步：换掉 NLP 引擎

先装中文模型：

```bash
pip install presidio-analyzer presidio-anonymizer
python -m spacy download zh_core_web_lg
```

`zh_core_web_lg` 是 spaCy 的中文大模型，能做中文分词、词性标注和命名实体识别。（它会带上自己的中文分词依赖，安装时间比英文模型长一些。）

然后用配置告诉 Presidio 用它：

```yaml
# zh-config.yml
nlp_engine_name: spacy
models:
  - lang_code: zh
    model_name: zh_core_web_lg

ner_model_configuration:
  # 把 spaCy 的标签映射成 Presidio 的实体类型
  model_to_presidio_entity_mapping:
    PERSON: PERSON
    PER: PERSON
    GPE: LOCATION
    LOC: LOCATION
    ORG: ORGANIZATION
    DATE: DATE_TIME
    TIME: DATE_TIME
  # 机构名这类容易误报的，分数打个折
  low_confidence_score_multiplier: 0.4
  low_score_entity_names:
    - ORGANIZATION
  default_score: 0.85
```

**`model_to_presidio_entity_mapping` 这一段是必须的**，很多人会漏。spaCy 吐出来的标签叫 `GPE`（地缘政治实体）、`PER` 之类，Presidio 内部用的是 `LOCATION`、`PERSON`——中间要有一张翻译表，否则模型认出来了，Presidio 也不认。

装配起来：

```python
from presidio_analyzer import AnalyzerEngine
from presidio_analyzer.nlp_engine import NlpEngineProvider

provider = NlpEngineProvider(conf_file="zh-config.yml")
nlp_engine = provider.create_engine()

analyzer = AnalyzerEngine(
    nlp_engine=nlp_engine,
    supported_languages=["zh"],
)

results = analyzer.analyze(text="张伟去了北京", language="zh")
print(results)
# 现在「张伟」能被识别为 PERSON，「北京」为 LOCATION
```

**注意 `analyze()` 里的 `language="zh"` 必须和配置里的 `lang_code` 对上**，写成 `"cn"` 会直接报错说不支持。

到这一步，人名、地名、机构名解决了。但身份证和手机号还是不认识——那些需要我们自己写识别器。

## 二、第二步：写一个手机号识别器

最简单的识别器是基于正则的 `PatternRecognizer`：

```python
from presidio_analyzer import Pattern, PatternRecognizer

phone_recognizer = PatternRecognizer(
    supported_entity="CN_PHONE_NUMBER",
    supported_language="zh",
    patterns=[
        Pattern(
            name="中国大陆手机号",
            regex=r"(?<!\d)1[3-9]\d{9}(?!\d)",
            score=0.6,
        )
    ],
    context=["手机", "电话", "联系方式", "号码", "手机号"],
)
```

### 这里有个中文特有的大坑

看清楚上面的正则：我用的是 `(?<!\d)...(?!\d)`，而**不是**更常见的 `\b1[3-9]\d{9}\b`。

原因是：**`\b` 在中文文本里会失效。**

`\b` 表示「单词字符和非单词字符的边界」。在 Python 里，`\w` 默认包含 Unicode 字母，**而汉字属于 Unicode 字母**。所以在「手机13800138000」这个字符串里：

* `机` 是 `\w`
* `1` 也是 `\w`
* **两者之间根本没有边界**，`\b` 匹配不上

```python
import re
re.search(r"\b1[3-9]\d{9}\b", "手机13800138000")      # None  ← 匹配失败
re.search(r"(?<!\d)1[3-9]\d{9}(?!\d)", "手机13800138000")  # 匹配成功
```

英文里 `phone 13800138000` 中间有空格，`\b` 工作正常，所以照抄英文教程的正则，在中文场景下会静默失效——**它不报错，只是什么都找不到**。这个坑我见过好几次。

**结论：中文正则里，用 `(?<!\d)` / `(?!\d)` 这类环视断言替代 `\b`。**

## 三、第三步：写一个带校验位的身份证识别器

手机号只能给 0.6 分，因为 11 位数字撞车的概率不低——订单号、流水号都可能长这样。

身份证不一样：**它有校验位，能算出真假**。这让识别可以做到相当高的置信度。

![身份证识别的三级跳](/assets/imgs/presidio-chinese-scoring.svg)

思路就是图里这三步：正则先粗筛（分数很低），校验位算一遍（分数提上来），再看看附近有没有「身份证」这类提示词（分数再提）。

实现上，继承 `PatternRecognizer` 并重写 `validate_result`：

```python
from typing import List, Optional
from presidio_analyzer import Pattern, PatternRecognizer


class CnIdCardRecognizer(PatternRecognizer):
    """中国大陆居民身份证号识别器（GB 11643-1999，18 位）。"""

    PATTERNS = [
        Pattern(
            name="身份证号（弱）",
            # 6 位地址码 + 8 位出生日期 + 3 位顺序码 + 1 位校验位
            regex=r"(?<![0-9Xx])\d{6}(?:19|20)\d{2}"
                  r"(?:0[1-9]|1[0-2])(?:0[1-9]|[12]\d|3[01])"
                  r"\d{3}[\dXx](?![0-9Xx])",
            score=0.1,          # 只是形状对，先给个很低的分
        )
    ]

    CONTEXT = ["身份证", "证件号", "身份证号", "证件号码", "身份证件"]

    # ISO 7064:1983 MOD 11-2 的加权因子
    WEIGHTS = (7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2)
    CHECK_CODES = "10X98765432"

    def __init__(
        self,
        patterns: Optional[List[Pattern]] = None,
        context: Optional[List[str]] = None,
        supported_language: str = "zh",
        supported_entity: str = "CN_ID_CARD",
    ) -> None:
        super().__init__(
            supported_entity=supported_entity,
            patterns=patterns or self.PATTERNS,
            context=context or self.CONTEXT,
            supported_language=supported_language,
        )

    def validate_result(self, pattern_text: str) -> Optional[bool]:
        """校验位算得对就是真的，算不对直接否掉。"""
        if len(pattern_text) != 18:
            return False

        body, check = pattern_text[:17], pattern_text[17].upper()
        if not body.isdigit():
            return False

        total = sum(int(d) * w for d, w in zip(body, self.WEIGHTS))
        return self.CHECK_CODES[total % 11] == check
```

`validate_result` 的返回值有三种含义，这个机制值得记住：

| 返回 | 含义 | 分数变化 |
| --- | --- | --- |
| `True` | 确认有效 | **提到接近满分** |
| `False` | 确认无效 | **直接丢弃这个结果** |
| `None` | 无法判断 | 保持正则给的原始分数 |

也就是说，**校验位算法不只是加分，更重要的是它能把误报直接杀掉**。一个 18 位的订单号即使形状对上了，校验位一算不对，`False` 一返回就没了。这比单纯调阈值干净得多。

验证一下：

```python
r = CnIdCardRecognizer()
print(r.validate_result("11010519491231002X"))  # True   ← 国标里的示例号
print(r.validate_result("110105194912310021"))  # False  ← 改了校验位
```

## 四、组装起来

```python
from presidio_analyzer import AnalyzerEngine, RecognizerRegistry
from presidio_analyzer.nlp_engine import NlpEngineProvider
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

# 1) 中文 NLP 引擎
nlp_engine = NlpEngineProvider(conf_file="zh-config.yml").create_engine()

# 2) 注册表：内置的 + 我们自己的
registry = RecognizerRegistry()
registry.load_predefined_recognizers(languages=["zh"], nlp_engine=nlp_engine)
registry.add_recognizer(phone_recognizer)
registry.add_recognizer(CnIdCardRecognizer())

# 3) 引擎
analyzer = AnalyzerEngine(
    registry=registry,
    nlp_engine=nlp_engine,
    supported_languages=["zh"],
)

text = "客户张伟，手机 13800138000，身份证 11010519491231002X，住北京市朝阳区"
results = analyzer.analyze(text=text, language="zh")
for r in results:
    print(r.entity_type, r.score, text[r.start:r.end])

# 4) 脱敏
anonymizer = AnonymizerEngine()
print(anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators={
        "CN_PHONE_NUMBER": OperatorConfig("mask", {
            "masking_char": "*", "chars_to_mask": 7, "from_end": False,
        }),
        "CN_ID_CARD": OperatorConfig("replace", {"new_value": "[身份证]"}),
        "PERSON": OperatorConfig("replace", {"new_value": "[某客户]"}),
        "DEFAULT": OperatorConfig("redact"),
    },
).text)
```

注意 `load_predefined_recognizers(languages=["zh"])`——**内置的邮箱、IP、信用卡这些识别器是语言无关的**（格式全世界一样），把它们一起加载进来能省不少事。真正需要自己写的，只有中国特有的那几种。

## 五、打进 Docker 镜像

上一篇讲过 .NET 团队应该用容器方式。但官方镜像里装的是英文模型，中文配置需要自己构建。

关键在 `presidio-analyzer/presidio_analyzer/conf/default.yaml`——**这个文件在 `docker build` 阶段会被读取，里面声明的模型会被自动下载进镜像**。把它改成中文配置，再构建自己的镜像即可：

```dockerfile
FROM ghcr.io/data-privacy-stack/presidio-analyzer:latest
# 放入你的中文配置和自定义识别器
COPY zh-config.yml /app/conf/zh-config.yml
COPY recognizers/ /app/recognizers/
```

自定义识别器的加载方式有两种：写代码在启动时 `add_recognizer()`，或者用 **YAML 无代码配置**（Presidio 支持从 YAML 文件加载识别器注册表）。后者对运维更友好——改个正则不用重新构建镜像。

还有一个更轻的办法：**ad-hoc 识别器**。调 `/analyze` 时在请求体里带上 `ad_hoc_recognizers` 字段，临时追加规则，完全不用碰镜像。适合规则变动频繁的场景，代价是每次请求都要传一遍。

## 六、必须说的几件事

### 1. 一定要自己测一遍

**不要因为跑通了一个示例就以为搞定了。** 中文 NER 的实际效果和你的文本类型强相关：

* 客服工单里的口语化人名（「小王」「张总」）——大概率认不出来
* 单字姓名、少数民族姓名——容易漏
* 「北京烤鸭」里的「北京」——会被当成地名误报

**做法**：从你自己的真实数据里抽 200 条，人工标一遍，跑一遍，算漏检率和误报率。这个工作量不大，但它是唯一能告诉你「能不能上生产」的东西。Presidio 官方也提供了评估工具的文档，可以参考。

### 2. 分数策略要跟着场景走

* **日志脱敏**：宁可误报，不能漏。阈值调低，误报无非是多打了几个占位符。
* **给业务展示的数据**：宁可漏，不能误报。把客户名字错误地涂掉，用户体验很糟。

**同一套识别器，两个场景应该用不同的 `score_threshold`。**

### 3. 还有一批实体可以照此扩展

同样的模式可以用在：

| 实体 | 校验方式 |
| --- | --- |
| **银行卡号** | Luhn 算法（Presidio 内置的信用卡识别器已经覆盖了大部分场景） |
| **统一社会信用代码** | 18 位，GB 32100-2015 定义了校验规则 |
| **车牌号** | 有固定格式，但没有校验位，只能靠正则加上下文 |
| **公司内部工号** | 你自己最清楚规则 |

**建议按「有没有校验位」排优先级做**：有校验位的先做，因为投入产出比最高——一段几行的算法就能把误报砍掉一大半。

### 4. 它依然不保证找全

第一篇里官方的那句警告，在中文场景下只会更严重，不会更轻：**中文 NER 的成熟度整体不如英文**。

**别把 Presidio 当成合规的终点。** 它是纵深防御里的一层——最该做的事情，仍然是从一开始就别把敏感数据写进不该写的地方。

## 总结

* 中文场景要改三处：**NLP 模型换中文、补中国特有的识别器、上下文提示词改成中文**。
* 配置 NLP 引擎时，**`model_to_presidio_entity_mapping` 那张翻译表不能漏**。
* **中文正则里 `\b` 会失效**（汉字也是 `\w`），用 `(?<!\d)` / `(?!\d)` 代替——这个坑不报错，只是静默找不到。
* `validate_result` 返回 `False` 会**直接丢弃结果**，所以校验位算法既是加分项，更是**杀误报的利器**。
* 部署时改 `conf/default.yaml` 构建自己的镜像；规则常变就用 **YAML 无代码配置**或 **ad-hoc 识别器**。
* **一定要拿自己的真实数据测一遍**，示例跑通不代表能上生产。

[下一篇](/2026/07/24/presidio-llm-pii-gateway/)是本系列最后一篇：**给大模型应用加一道脱敏闸门**——用户把客户资料贴进 AI 助手，怎么保证真实姓名和手机号永远不会离开你的机房。

## 延伸阅读

* [Presidio 多语言支持文档](https://data-privacy-stack.github.io/presidio/analyzer/languages/)
* [开发自定义识别器](https://data-privacy-stack.github.io/presidio/analyzer/adding_recognizers/)
* [从 .NET 调用 Presidio（本系列上篇）](/2026/07/10/call-presidio-from-dotnet/)
