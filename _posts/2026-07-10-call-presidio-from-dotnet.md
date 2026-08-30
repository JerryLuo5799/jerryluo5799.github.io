---
layout: post
title:  "Presidio 实战（二）：从 ASP.NET Core 调用 Presidio"
date:   2026-07-10 09:30:00 +0800--
categories: [AI]
tags: [Presidio]
---

## 前言

[Presidio](/2026/07/03/what-is-presidio/) 是一套能自动识别并脱敏 PII 的开源工具。但它有一个对我们来说很现实的问题：

**它是纯 Python 的。**

我把整个仓库搜了一遍，`.cs` 文件数量是 **0**，官方文档里也没有任何 .NET 相关的章节。Java、Node 的示例同样没有——它就是一个 Python 项目。

那 .NET 团队还能用吗？能，而且不难。这篇讲清楚怎么接。

## 一、三种接法，选哪个

| 方案 | 怎么做 | 评价 |
| --- | --- | --- |
| **REST 服务** | Docker 起容器，HTTP 调用 | ✅ **推荐** |
| Python.NET | 进程内嵌 Python 运行时 | ❌ 部署地狱 |
| 起子进程 | `Process.Start` 调 Python 脚本 | ❌ 性能和错误处理都很糟 |

**直接选 REST。** 理由：

* **部署、扩容、升级完全独立**——脱敏服务扛不住了单独加副本，跟你的应用无关。
* **故障隔离**——Python 侧 OOM 或崩溃，你的进程不会跟着倒下。
* **依赖干净**——你的构建产物里不会混进 500MB 的 NLP 模型。

后两种方案的唯一优势是省了一次网络往返，但为此要把 Python 运行时和 spaCy 模型塞进 .NET 的部署包里，代价完全不成比例。

![.NET 集成架构](/assets/imgs/presidio-dotnet-architecture.svg)

**注意图里最重要的一条：一次脱敏是两次调用。** 先问 analyzer「哪里有 PII」，拿到实体清单，再把「原文 + 这份清单 + 处理规则」一起交给 anonymizer。

很多人第一次用会困惑：「为什么不能一步到位？」因为这个拆分是有意的——识别结果在中间是**可以被你干预**的。后面会讲到实际用途。

## 二、把服务跑起来

```yaml
# docker-compose.yml
services:
  presidio-analyzer:
    image: ghcr.io/data-privacy-stack/presidio-analyzer:latest
    ports:
      - "5002:3000"     # 容器内默认监听 3000
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      start_period: 90s  # 要加载 NLP 模型，给足时间
      retries: 3
    restart: unless-stopped

  presidio-anonymizer:
    image: ghcr.io/data-privacy-stack/presidio-anonymizer:latest
    ports:
      - "5001:3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      start_period: 15s
      retries: 3
    restart: unless-stopped
```

两个细节值得说：

1. **容器内的默认端口是 3000**，不是 5001/5002。宿主机端口是官方文档习惯用的，别搞混。想改容器内端口，设 `PORT` 环境变量。
2. **analyzer 的 `start_period` 要给到 90 秒**。它启动时加载 spaCy 模型，冷启动十几秒很正常。这个值给小了，编排系统会误判成「没起来」然后反复重启——一个非常常见又非常费解的故障。

```bash
docker compose up -d
curl http://localhost:5002/health
```

## 三、定义数据契约

Presidio 的 JSON 用的是蛇形命名（`entity_type`、`score_threshold`），跟 .NET 的习惯不一样。我倾向于**显式标注 `JsonPropertyName`**，而不是全局配一个命名策略——契约是别人定的，写死更不容易出意外。

```csharp
using System.Text.Json.Serialization;

// ---------- POST /analyze 的请求 ----------
public sealed class AnalyzeRequest
{
    [JsonPropertyName("text")]
    public required string Text { get; init; }

    [JsonPropertyName("language")]
    public required string Language { get; init; }

    // 只找这几类实体。留空表示全找，但会明显变慢。
    [JsonPropertyName("entities")]
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string[]? Entities { get; init; }

    // 低于这个分数的结果直接丢弃。
    [JsonPropertyName("score_threshold")]
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public double? ScoreThreshold { get; init; }
}

// ---------- analyzer 返回的每一条 ----------
public sealed class AnalyzerResult
{
    [JsonPropertyName("entity_type")]
    public required string EntityType { get; init; }

    [JsonPropertyName("start")]
    public int Start { get; init; }

    [JsonPropertyName("end")]
    public int End { get; init; }

    [JsonPropertyName("score")]
    public double Score { get; init; }
}
```

anonymizer 这一侧，处理规则的 JSON 形状是这样的：

```json
{
  "type": "mask",
  "masking_char": "*",
  "chars_to_mask": 4,
  "from_end": true
}
```

**注意操作类型放在 `type` 字段里，参数和它平级铺开**，不是嵌套在一个 `params` 对象里的。这个设计有点特别，用一个带 `JsonExtensionData` 的类来对付它最省事：

```csharp
public sealed class OperatorConfig
{
    [JsonPropertyName("type")]
    public required string Type { get; init; }

    // 各 operator 自己的参数，序列化时会平铺到同一层
    [JsonExtensionData]
    public Dictionary<string, object>? Params { get; init; }
}

public sealed class AnonymizeRequest
{
    [JsonPropertyName("text")]
    public required string Text { get; init; }

    [JsonPropertyName("analyzer_results")]
    public required IReadOnlyList<AnalyzerResult> AnalyzerResults { get; init; }

    // key 是实体类型，或者 "DEFAULT" 表示兜底规则
    [JsonPropertyName("anonymizers")]
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public Dictionary<string, OperatorConfig>? Anonymizers { get; init; }
}

public sealed class AnonymizeResponse
{
    [JsonPropertyName("text")]
    public required string Text { get; init; }

    [JsonPropertyName("items")]
    public IReadOnlyList<AnonymizedItem> Items { get; init; } = [];
}

public sealed class AnonymizedItem
{
    [JsonPropertyName("entity_type")] public required string EntityType { get; init; }
    [JsonPropertyName("start")]       public int Start { get; init; }
    [JsonPropertyName("end")]         public int End { get; init; }
    [JsonPropertyName("text")]        public string? Text { get; init; }
    [JsonPropertyName("operator")]    public string? Operator { get; init; }
}
```

## 四、封装成一个服务

```csharp
public interface IPiiScrubber
{
    Task<string> ScrubAsync(string text, CancellationToken ct = default);
}

internal sealed class PresidioScrubber(
    IHttpClientFactory httpClientFactory,
    ILogger<PresidioScrubber> logger) : IPiiScrubber
{
    // 默认规则：手机号遮蔽前 7 位，人名换成占位符，其余一律换成类型名
    private static readonly Dictionary<string, OperatorConfig> DefaultOperators = new()
    {
        ["PHONE_NUMBER"] = new OperatorConfig
        {
            Type = "mask",
            Params = new Dictionary<string, object>
            {
                ["masking_char"] = "*",
                ["chars_to_mask"] = 7,
                ["from_end"] = false,
            },
        },
        ["PERSON"] = new OperatorConfig
        {
            Type = "replace",
            Params = new Dictionary<string, object> { ["new_value"] = "[某用户]" },
        },
        ["DEFAULT"] = new OperatorConfig { Type = "replace" },
    };

    public async Task<string> ScrubAsync(string text, CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(text))
            return text;

        // 第一步：找
        var analyzer = httpClientFactory.CreateClient("presidio-analyzer");
        var analyzeResponse = await analyzer.PostAsJsonAsync("/analyze",
            new AnalyzeRequest
            {
                Text = text,
                Language = "en",
                ScoreThreshold = 0.5,
            }, ct);

        analyzeResponse.EnsureSuccessStatusCode();
        var findings = await analyzeResponse.Content
            .ReadFromJsonAsync<List<AnalyzerResult>>(ct) ?? [];

        // 什么都没找到，就别白跑第二趟了
        if (findings.Count == 0)
            return text;

        logger.LogDebug("Presidio 命中 {Count} 个实体，类型：{Types}",
            findings.Count,
            string.Join(", ", findings.Select(f => f.EntityType).Distinct()));

        // 第二步：改
        var anonymizer = httpClientFactory.CreateClient("presidio-anonymizer");
        var anonymizeResponse = await anonymizer.PostAsJsonAsync("/anonymize",
            new AnonymizeRequest
            {
                Text = text,
                AnalyzerResults = findings,
                Anonymizers = DefaultOperators,
            }, ct);

        anonymizeResponse.EnsureSuccessStatusCode();
        var result = await anonymizeResponse.Content
            .ReadFromJsonAsync<AnonymizeResponse>(ct);

        return result?.Text ?? text;
    }
}
```

注册（.NET 8 及以上）：

```csharp
// Program.cs
builder.Services.AddHttpClient("presidio-analyzer", client =>
    {
        client.BaseAddress = new Uri(builder.Configuration["Presidio:AnalyzerUrl"]!);
        client.Timeout = TimeSpan.FromSeconds(30);   // 冷启动会慢，别设太短
    })
    .AddStandardResilienceHandler();                 // 重试 + 熔断 + 超时

builder.Services.AddHttpClient("presidio-anonymizer", client =>
    {
        client.BaseAddress = new Uri(builder.Configuration["Presidio:AnonymizerUrl"]!);
        client.Timeout = TimeSpan.FromSeconds(10);
    })
    .AddStandardResilienceHandler();

builder.Services.AddSingleton<IPiiScrubber, PresidioScrubber>();
```

`AddStandardResilienceHandler()` 来自 `Microsoft.Extensions.Http.Resilience` 包，一行就把重试、熔断、超时全配齐了，比自己拼策略省事得多。

用起来：

```csharp
app.MapPost("/tickets", async (TicketDto dto, IPiiScrubber scrubber, ILogger<Program> log) =>
{
    var safe = await scrubber.ScrubAsync(dto.Description);
    log.LogInformation("收到工单：{Description}", safe);
    return Results.Ok();
});
```

## 五、几个真实的坑

### 1. 千万别把容器暴露到公网

**这两个容器没有任何鉴权机制**——没有 API Key，没有 OAuth，谁都能调。

它们的定位就是内网服务。放在 Docker 网络里、K8s 集群内部、或者绑在 `127.0.0.1` 上，**绝不能配个公网域名扔出去**。真要跨网络访问，前面加一层你自己的网关。

### 2. 偏移量是「码点」，不是 UTF-16 单元

Presidio 返回的 `start` / `end` 是 **Python 的字符串索引，也就是 Unicode 码点**；而 C# 的 `string` 是 **UTF-16**。

对绝大多数文本（包括中文汉字）这两者一致，但**碰到 emoji 这类需要代理对表示的字符，就会错位**：

```csharp
var text = "🎉 联系我 13800138000";
// Python 那边看到的长度：14
// C# 的 text.Length：15   ← 🎉 占了两个 UTF-16 单元
```

**只要你老老实实把偏移量原样传回给 anonymizer，就完全不受影响**——字符串是 Python 那边自己切的。这个坑只在你想在 C# 侧**自己按偏移量截取字符串**时才会踩到。如果非要这么做，用 `StringInfo` 或者先转成码点数组，别直接 `Substring`。

### 3. 一次传一批，别一条条循环

`text` 字段**可以传数组**，服务端会批量处理：

```json
{"text": ["第一条文本", "第二条文本"], "language": "en"}
```

返回的也是数组的数组，顺序一一对应。批量比循环单发快得多——省下来的是网络往返，不是计算。

### 4. 用 `entities` 缩小搜索范围

不指定 `entities` 意味着**跑全部内置识别器**，几十个。如果你只关心手机号和邮箱：

```csharp
Entities = ["PHONE_NUMBER", "EMAIL_ADDRESS"],
```

这一行能带来相当可观的性能提升，而且**顺带减少误报**——不去找的东西自然不会认错。

### 5. 别忘了它是「尽力而为」

再强调一遍官方那句警告：**Presidio 不保证找全**。

所以在日志脱敏这类场景里，它是一层补救，不是免死金牌。**真正该做的还是从源头上别把整个用户对象往日志里塞。**

## 六、中间那一步能干什么

回到开头的问题：为什么是两次调用，而不是一个接口搞定？

因为**识别结果在中间是你可以动手脚的**。几个实际用途：

**审计**——把命中记录下来，你就能回答「上个月的日志里到底出现过多少次手机号」。这在合规检查时非常有用。

**白名单**——公司自己的客服热线被识别成 `PHONE_NUMBER`，但它根本不是隐私。在中间过滤掉：

```csharp
var hotline = new[] { "400-820-8820" };
findings = findings
    .Where(f => !hotline.Contains(text[f.Start..f.End]))
    .ToList();
```

**分级处理**——高分的直接脱敏，低分的（0.5 到 0.7）不动，但打个标记进人工复核队列。这比一刀切的阈值精细得多。

**补充自己的规则**——公司内部的工号、合同编号，Presidio 不认识，你可以在这一步自己用正则找出来，追加进 `findings` 再一起交给 anonymizer 处理。

## 总结

* **走 REST**，不要在 .NET 进程里嵌 Python。部署、扩容、故障隔离全都更干净。
* **一次脱敏两次调用**：analyzer 找、anonymizer 改，中间那一步是留给你的。
* 契约上注意：**处理规则的 `type` 和参数是平级的**，用 `JsonExtensionData` 接最省事。
* 配置上注意：**容器内端口是 3000**、**analyzer 冷启动要给到 90 秒的 `start_period`**、**超时别设太短**。
* 安全上注意：**这两个容器没有鉴权，只能待在内网**。
* 用 `entities` 白名单和 `score_threshold` 同时优化性能和准确率。

还有一个更棘手的问题在等着：**默认配置对中文基本无效**——`en_core_web_lg` 连中文的词都切不对，更别说认出人名和身份证号了。[让 Presidio 认识中文](/2026/07/17/presidio-chinese-pii/)里讲了怎么自己动手解决。

## 延伸阅读

* [Presidio 安装文档](https://data-privacy-stack.github.io/presidio/installation/?wt.mc_id=MVP_324329)
* [Presidio REST API 参考](https://data-privacy-stack.github.io/presidio/api-docs/api-docs.html?wt.mc_id=MVP_324329)
* [Presidio 入门](/2026/07/03/what-is-presidio/)
