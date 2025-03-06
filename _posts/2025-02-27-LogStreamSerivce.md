---
layout: post
title:  "WebAPI借助Server-Sent Events (SSE)实现日志流式输出"
date:   2025-02-27 21:42:55 +0800--
categories: [.NET]
tags: [流式日志输出, JSONPath]  
---

### 1. 前言

当我们调用一个需要长时间执行的接口的时候, 例如导入, 导出接口, 如果遇到问题, 排查会非常的繁琐。 那么, 有没有可能让我们的Web应用， 可以和控制台应用一样，实时的输出日志呢？ 我们可以借助Server-Sent Events (SSE)的方式实现。

假设我们有一个前后端分离的项目 (.NET8 + React), 同时实现了导入和导出的两个接口, 
如何实现日志流式输出到前端React页面呢?

### 2. 总体思路

- 后端使用Server-Sent Events (SSE) 技术推送日志流
- 前端通过EventSource API接收并实时显示
- 重定向.NET的Console输出到SSE通道
- 通过不同通道(channel)控制并发任务的输出


### 3. 代码实现

### 3.1 通用日志服务

```csharp
// LogService.cs
public interface ILogService
{
    string CreateLogChannel();
    void RemoveLogChannel(string channelId);
    Task WriteLogAsync(string channelId, string log);
    IAsyncEnumerable<string> StreamLogsAsync(string channelId, CancellationToken ct);
}

public class LogService : ILogService
{
    private readonly ConcurrentDictionary<string, Channel<string>> _channels = new();
    private readonly TimeSpan _channelTimeout = TimeSpan.FromMinutes(30);

    public string CreateLogChannel()
    {
        var channelId = Guid.NewGuid().ToString();
        var channel = Channel.CreateUnbounded<string>();
        _channels.TryAdd(channelId, channel);
        
        // 设置通道超时清理
        _ = Task.Delay(_channelTimeout).ContinueWith(_ => 
            RemoveLogChannel(channelId));
            
        return channelId;
    }

    public void RemoveLogChannel(string channelId)
    {
        if (_channels.TryRemove(channelId, out var channel))
        {
            channel.Writer.Complete();
        }
    }

    public async Task WriteLogAsync(string channelId, string log)
    {
        if (_channels.TryGetValue(channelId, out var channel))
        {
            await channel.Writer.WriteAsync(log);
        }
    }

    public IAsyncEnumerable<string> StreamLogsAsync(string channelId, CancellationToken ct)
    {
        return _channels.TryGetValue(channelId, out var channel) 
            ? channel.Reader.ReadAllAsync(ct) 
            : AsyncEnumerable.Empty<string>();
    }
}
```

### 3.2 通用日志控制器

```csharp
[ApiController]
[Route("api/[controller]")]
public class LogStreamController : ControllerBase
{
    private readonly ILogService _logService;

    public LogStreamController(ILogService logService)
    {
        _logService = logService;
    }

    [HttpGet("{channelId}")]
    public async Task StreamLogs(string channelId, CancellationToken ct)
    {
        Response.Headers.Add("Content-Type", "text/event-stream");
        Response.Headers.Add("Cache-Control", "no-cache");
        Response.Headers.Add("Connection", "keep-alive");

        try
        {
            await foreach (var log in _logService.StreamLogsAsync(channelId, ct))
            {
                var sseEvent = $"data: {log}\n\n";
                await Response.WriteAsync(sseEvent, ct);
                await Response.Body.FlushAsync(ct);
            }
        }
        catch (OperationCanceledException)
        {
            // 客户端断开连接
        }
    }
}
```

### 3.3 Console.WriteLine 拦截器（支持多通道）

```csharp
// LogInterceptor.cs
public class LogInterceptor : TextWriter
{
    private readonly ILogService _logService;
    private static readonly AsyncLocal<string> _currentChannel = new();
    private readonly TextWriter _originalOut;

    public override Encoding Encoding => Encoding.UTF8;

    public LogInterceptor(ILogService logService)
    {
        _logService = logService;
        _originalOut = Console.Out;
    }

    public static void SetCurrentChannel(string channelId)
    {
        _currentChannel.Value = channelId;
    }

    public override void WriteLine(string? value)
    {
        base.WriteLine(value);
        _originalOut.WriteLine(value); // 保持原控制台输出
        
        if (!string.IsNullOrEmpty(_currentChannel.Value))
        {
            _ = _logService.WriteLogAsync(_currentChannel.Value, value ?? "");
        }
    }
}

// Program.cs 注册
builder.Services.AddSingleton<ILogService, LogService>();
var logService = builder.Services.BuildServiceProvider().GetService<ILogService>();
Console.SetOut(new LogInterceptor(logService));
```

### 3.4 导出服务

```csharp
public class ExportService
{
    private readonly ILogService _logService;

    public ExportService(ILogService logService)
    {
        _logService = logService;
    }

    public async Task ExportDataAsync()
    {
        var channelId = _logService.CreateLogChannel();
        LogInterceptor.SetCurrentChannel(channelId);

        try
        {
            Console.WriteLine("开始导出数据...");
            
            for (int i = 1; i <= 100; i++)
            {
                await Task.Delay(100);
                Console.WriteLine($"正在导出第 {i} 条记录");
            }

            Console.WriteLine("导出完成！");
        }
        finally
        {
            LogInterceptor.SetCurrentChannel(null);
            _logService.RemoveLogChannel(channelId);
        }
    }
}
```

### 3.4 导出接口

```csharp
[ApiController]
[Route("api/[controller]")]
public class ExportController : ControllerBase
{
    private readonly ExportService _exportService;

    public ExportController(ExportService exportService)
    {
        _exportService = exportService;
    }

    [HttpPost]
    public async Task<IActionResult> StartExport()
    {
        // 异步启动导出任务
        _ = Task.Run(async () => await _exportService.ExportDataAsync());
        return Accepted();
    }
}
```

### 3.5 前端日志组件（React）

```javascript
// LogStream.tsx
import { useState, useEffect } from 'react';

interface LogStreamProps {
  channelId: string;
  onClose?: () => void;
}

const LogStream = ({ channelId, onClose }: LogStreamProps) => {
  const [logs, setLogs] = useState<string[]>([]);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!channelId) return;

    const eventSource = new EventSource(`/api/LogStream/${channelId}`);

    const handleMessage = (e: MessageEvent) => {
      setLogs(prev => [...prev, e.data]);
      autoScroll();
    };

    const autoScroll = () => {
      const container = document.getElementById('log-container');
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    };

    eventSource.onopen = () => setIsConnected(true);
    eventSource.onmessage = handleMessage;
    eventSource.onerror = () => {
      setIsConnected(false);
      onClose?.();
    };

    return () => {
      eventSource.close();
    };
  }, [channelId, onClose]);

  return (
    <div className="log-stream">
      <div className="status">
        日志通道：{channelId} | 状态：{isConnected ? '已连接' : '已断开'}
      </div>
      <div id="log-container" className="logs">
        {logs.map((log, i) => (
          <div key={i} className="log-line">{log}</div>
        ))}
      </div>
    </div>
  );
};

export default LogStream;
```

### 3.6 前端调用（React + TypeScript）

```tsx
// ExportPage.tsx
import { useState } from 'react';
import LogStream from './LogStream';

const ExportPage = () => {
  const [logChannel, setLogChannel] = useState<string | null>(null);

  const startExport = async () => {
    const response = await fetch('/api/Export', { method: 'POST' });
    const { channelId } = await response.json();
    setLogChannel(channelId);
  };

  return (
    <div>
      <button onClick={startExport}>开始导出</button>
      {logChannel && (
        <LogStream 
          channelId={logChannel}
          onClose={() => setLogChannel(null)}
        />
      )}
    </div>
  );
};
```

### 4. 方案特点

- 完全解耦设计
  - 日志服务与业务逻辑分离
  - 前端组件与具体业务无关
  - 通道生命周期自动管理
  - 多任务并发支持

- 每个任务独立日志通道
  - 支持同时查看多个任务日志
  - 通道ID隔离确保数据安全

- 无缝集成现有代码
  - 通过Console.WriteLine自动捕获
  - 无需修改原有日志输出代码
  - 保持控制台原始输出不变

- 扩展性强
  - 支持添加日志过滤、格式化等中间件
  - 可扩展支持WebSocket协议
  - 轻松集成日志持久化功能

### 5. 潜在改进功能

#### 5.1 日志格式化

```csharp
public class FormattedLogInterceptor : LogInterceptor
{
    public override void WriteLine(string? value)
    {
        var formattedLog = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] {value}";
        base.WriteLine(formattedLog);
    }
}
```

#### 5.2 日志格式化

```csharp
// 在日志服务中添加权限验证
public IAsyncEnumerable<string> StreamLogsAsync(string channelId, CancellationToken ct)
{
    if (!CheckAccessPermission(channelId))
    {
        throw new UnauthorizedAccessException();
    }
    // ...原有代码
}
```

#### 5.3 日志格式化

```csharp
public class LogService : ILogService
{
    private readonly ConcurrentDictionary<string, (Channel<string>, DateTime)> _channels = new();
    
    public IEnumerable<ChannelInfo> GetActiveChannels()
    {
        return _channels.Select(x => new ChannelInfo(
            x.Key,
            x.Value.Item1.Reader.Count,
            x.Value.Item2
        ));
    }
}
```