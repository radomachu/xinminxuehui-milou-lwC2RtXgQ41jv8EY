## 前言

在AI快速发展的今天，微软推出了多个AI开发框架，从早期的AutoGen到Semantic Kernel，再到最新的Microsoft Agent Framework。很多开发者可能会有疑问：为什么微软要推出这么多框架？它们之间有什么区别？本文将通过一个实际的AI美女聊天群组项目，带你深入理解Microsoft Agent Framework，掌握多智能体开发的核心概念。

本文的示例代码已开源：[agent-framework-tutorial-code/agent-groupchat](https://github.com)

![效果截图](https://img2024.cnblogs.com/blog/1690009/202510/1690009-20251028221439575-1520228988.jpg)

* [视频演示](https://github.com)

## 为什么微软要推出Microsoft Agent Framework？

### AutoGen vs Semantic Kernel vs Agent Framework

在讲解新框架之前，我们先理解一下微软AI框架的演进路径：

**AutoGen（研究导向）**

* 最早期的多智能体研究框架
* 侧重学术研究和实验性功能
* Python为主，生态相对独立

**Semantic Kernel（应用导向）**

* 面向生产环境的AI应用开发框架
* 强大的插件系统和内存管理
* 多语言支持（C#、Python、Java）
* 适合单一智能体应用

**Microsoft Agent Framework（企业导向）**

* 专为多智能体协作设计
* 内置工作流编排能力(Sequential、Concurrent、Handoff、GroupChat)
* 支持Handoff转移模式和GroupChat管理模式
* 与Azure AI Foundry深度集成
* 同时支持.NET和Python

### Agent Framework的核心优势

1. **原生多智能体支持**：无需手动管理智能体间的通信,框架自动处理消息路由
2. **声明式工作流**：通过`AgentWorkflowBuilder`构建复杂协作场景
3. **内置编排模式**：
   * **Handoff模式**：智能体通过function calling实现控制权转移
   * **GroupChat模式**：通过`GroupChatManager`选择下一个发言智能体(支持RoundRobin、Prompt-based等策略)
4. **状态管理**：支持checkpoint存储,可恢复中断的工作流

## 大模型基础知识科普

在使用框架之前，我们需要理解大模型的工作原理。很多开发者对大模型有神秘感，其实它本质上就是一个HTTP API调用。

### LLM API的本质

让我们用curl演示一个最简单的OpenAI API调用：

```
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "你好，请介绍一下自己"
      }
    ]
  }'
```

响应结果：

```
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1677652288,
  "model": "gpt-4o-mini",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "你好！我是一个AI助手，可以回答问题、提供建议..."
    },
    "finish_reason": "stop"
  }]
}
```

**关键点**：

1. LLM就是一个普通的HTTP接口
2. 输入：对话历史（messages数组）
3. 输出：AI生成的回复（content字段）
4. 所有复杂的Agent功能都是框架基于这个简单API构建的

### 函数调用（Function Calling）

函数调用是让LLM能够操作外部工具的关键机制。

**工作流程**：

1. 开发者定义可用的函数（工具）
2. LLM根据用户意图决定调用哪个函数
3. 框架执行函数并获取结果
4. 将结果返回给LLM继续对话

示例 - 定义天气查询函数：

```
{
  "name": "get_weather",
  "description": "查询指定城市的天气信息",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称，例如：北京"
      }
    },
    "required": ["city"]
  }
}
```

LLM的调用响应：

```
{
  "role": "assistant",
  "content": null,
  "function_call": {
    "name": "get_weather",
    "arguments": "{\"city\": \"北京\"}"
  }
}
```

**重点理解**：

* LLM不会直接执行函数，只是"建议"调用
* 框架负责解析并执行函数
* 执行结果需要再次发送给LLM才能生成最终回复

### MCP（Model Context Protocol）

MCP是新兴的标准化协议，用于LLM与外部工具的通信。

**MCP的优势**：

1. **标准化接口**：不同工具遵循统一协议
2. **动态工具发现**：运行时加载工具
3. **安全隔离**：工具在独立进程运行

在我们的示例项目中，使用MCP集成了阿里云通义万相图片生成能力。

## agent-groupchat项目解析

### 项目架构

项目采用.NET Aspire编排,前后端分离架构:

```
agent-groupchat/
├── AgentGroupChat.AppHost/         # Aspire编排入口
│   └── Program.cs                  # 服务编排配置
│
├── AgentGroupChat.AgentHost/       # 后端API服务（.NET 9）
│   ├── Services/
│   │   ├── AgentChatService.cs     # 核心聊天服务
│   │   ├── WorkflowManager.cs      # 工作流管理
│   │   ├── AgentRepository.cs      # 智能体配置管理
│   │   └── AgentGroupRepository.cs # 群组管理
│   ├── Models/
│   │   ├── AgentProfile.cs         # 智能体模型
│   │   └── AgentGroup.cs           # 群组模型
│   └── Program.cs                  # API端点
│
├── AgentGroupChat.Web/             # Blazor WebAssembly前端
│   ├── Components/
│   │   ├── Pages/
│   │   │   ├── Home.razor          # 聊天主页面
│   │   │   └── Admin.razor         # 管理后台
│   │   └── Layout/
│   │       └── MainLayout.razor    # 主布局
│   ├── Services/
│   │   └── AgentHostClient.cs      # API客户端
│   └── Program.cs                  # 前端入口
│
└── AgentGroupChat.ServiceDefaults/ # 共享服务配置
    └── Extensions.cs               # OpenTelemetry/健康检查
```

### Aspire编排说明

**什么是.NET Aspire？**

.NET Aspire是微软推出的云原生应用编排框架,简化分布式应用的开发和部署：

* **服务发现**：自动解析服务地址,前端无需硬编码API地址
* **统一启动**：一个命令启动所有服务
* **可观测性**：内置OpenTelemetry遥测数据收集
* **Dashboard**：实时查看服务状态、日志、指标

**AppHost配置**（`AgentGroupChat.AppHost/Program.cs`）：

```
var builder = DistributedApplication.CreateBuilder(args);

// 添加后端API服务
var agentHost = builder.AddProject("agenthost");

// 添加Blazor前端,引用后端服务
builder.AddProject("webfrontend")
    .WithExternalHttpEndpoints()  // 暴露外部访问端口
    .WithReference(agentHost)      // 注入agenthost服务发现信息
    .WaitFor(agentHost);           // 等待后端启动完成

builder.Build().Run();
```

**服务发现原理**：

前端通过Aspire自动获取后端地址（`Program.cs`）：

```
// Web项目的Program.cs
var agentHostUrl = builder.Configuration["AgentHostUrl"] ?? "https://localhost:7390";
builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(agentHostUrl) });
```

Aspire会自动将`agenthost`服务的实际地址注入到配置中。

### 智能体定义

项目创建了6个性格各异的AI美女角色，组成"AI世界公馆"：

**艾莲 (Elena)**

```
new PersistedAgentProfile
{
    Id = "elena",
    Name = "艾莲",
    Avatar = "🧠",
    SystemPrompt = "你是艾莲，一位来自巴黎的人文学者，专注于哲学、艺术和文学研究...",
    Description = "巴黎研究员，擅长哲学、艺术与思辨分析",
    Personality = "理性、深邃，喜欢引经据典，用哲学视角看世界"
}
```

**莉子 (Rina)**

```
new PersistedAgentProfile
{
    Id = "rina",
    Name = "莉子",
    Avatar = "🎮",
    SystemPrompt = "你是莉子，来自东京的元气少女，热爱动漫、游戏和可爱的事物...",
    Description = "东京元气少女，热爱动漫、游戏和可爱事物",
    Personality = "活泼、热情，说话带感叹号，喜欢用可爱的emoji"
}
```

其他角色：

* **克洛伊 (Chloe)**：纽约科技极客
* **安妮 (Annie)**：洛杉矶时尚博主
* **苏菲 (Sophie)**：伦敦哲学诗人

### 智能路由实现

这是项目的核心亮点 - Triage Agent自动将用户消息路由到最合适的AI角色。

**Triage Agent配置**：

```
var triageSystemPrompt = @"你是AI世界公馆的智能路由系统。

【核心规则】
1. 永远不要生成文本回复 - 你对用户完全透明
2. 立即调用handoff函数，不需要解释
3. 不要确认、问候或回应 - 只默默路由

【路由策略】
1. **直接提及**：用户用 @ 提到角色名，立即路由到该角色
2. **话题匹配**：
   - 哲学/艺术/文学 → 艾莲
   - 动漫/游戏/萌文化 → 莉子
   - 科技/编程/AI → 克洛伊
   - 时尚/美妆/生活 → 安妮
   - 诗歌/文学/情感 → 苏菲
3. **语气风格**：活泼→莉子，理性→艾莲，冷静→克洛伊
4. **上下文连贯**：查看对话历史，如果上一条是某专家回复且话题相关，继续路由到该专家

示例：
- ""@莉子 推荐动漫"" → handoff_to_rina
- ""如何学习机器学习？"" → handoff_to_chloe
- ""最新的时尚趋势是什么？"" → handoff_to_annie
";
```

**Handoff实现**：

```
// 创建Handoff工作流
var workflow = _workflowManager.GetOrCreateWorkflow(groupId);

// 运行工作流
await using StreamingRun run = await InProcessExecution.StreamAsync(workflow, messages);
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));

// 监听事件流
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is AgentRunUpdateEvent agentUpdate)
    {
        // 检测到specialist agent执行
        if (agentUpdate.ExecutorId != "triage")
        {
            var profile = _agentRepository.Get(agentUpdate.ExecutorId);
            
            // 提取LLM生成的文本
            var textContent = agentUpdate.Update.Contents
                .OfType()
                .FirstOrDefault();
            
            // 构建响应
            summaries.Add(new ChatMessageSummary
            {
                AgentId = agentUpdate.ExecutorId,
                AgentName = profile?.Name,
                AgentAvatar = profile?.Avatar,
                Content = textContent?.Text,
                IsUser = false
            });
        }
    }
}
```

### 关键技术点

**1. 动态智能体加载**

智能体配置存储在LiteDB中，支持运行时动态更新：

```
public class AgentRepository
{
    public List GetAllEnabled() 
    {
        return _collection
            .Find(a => a.Enabled)
            .ToList();
    }
    
    public void Upsert(PersistedAgentProfile agent) 
    {
        _collection.Upsert(agent);
    }
}
```

**2. 工作流管理**

每个智能体组有独立的工作流实例：

```
public class WorkflowManager
{
    private readonly Dictionary<string, Workflow> _workflows = new();
    
    public Workflow GetOrCreateWorkflow(string groupId)
    {
        if (!_workflows.TryGetValue(groupId, out var workflow))
        {
            var group = _groupRepository.Get(groupId);
            workflow = BuildHandoffWorkflow(group);
            _workflows[groupId] = workflow;
        }
        return workflow;
    }
}
```

**3. 消息持久化**

使用LiteDB存储会话历史：

```
public class PersistedSessionService
{
    public void AddMessage(string sessionId, ChatMessageSummary message)
    {
        var doc = new BsonDocument
        {
            ["SessionId"] = sessionId,
            ["AgentId"] = message.AgentId,
            ["Content"] = message.Content,
            ["Timestamp"] = message.Timestamp,
            ["IsUser"] = message.IsUser
        };
        
        _messagesCollection.Insert(doc);
    }
}
```

## 国内用户运行指南

### 方式一：使用阿里云百炼平台

![img](https://img2024.cnblogs.com/blog/1690009/202510/1690009-20251028222223856-1864861733.png)

1. **获取API密钥**

访问 [阿里云百炼](https://github.com)，创建应用并获取API Key。

2. **配置appsettings.json**

```
{
  "DefaultModelProvider": "OpenAI",
  "OpenAI": {
    "BaseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "ModelName": "qwen-plus",
    "ApiKey": "sk-your-api-key"
  }
}
```

3. **配置MCP生图**

需要开通MCP生图服务

使用的key也是和百炼的模型key一致

![img](https://img2024.cnblogs.com/blog/1690009/202510/1690009-20251028222517138-409894711.png)

```
{
  "McpServers": {
    "Servers": [
      {
        "Id": "dashscope-text-to-image",
        "Name": "DashScope Text-to-Image",
        "Endpoint": "https://dashscope.aliyuncs.com/api/v1/mcps/TextToImage/sse",
        "AuthType": "Bearer",
        "BearerToken": "",
        "TransportMode": "Sse",
        "Enabled": true,
        "Description": "阿里云 DashScope 文生图服务，用于生成图像"
      }
    ]
  }
}
```

4. **运行项目**

**方式A：使用Aspire一键启动（推荐）**

```
cd agent-groupchat
dotnet run --project AgentGroupChat.AppHost
```

Aspire Dashboard会自动打开（`http://localhost:15220`），显示：

* **agenthost**：后端API服务
* **webfrontend**：Blazor前端

直接点击webfrontend的URL即可访问应用。

**方式B：独立启动各服务**

终端1 - 启动后端：

```
cd agent-groupchat/AgentGroupChat.AgentHost
dotnet run
# 记下端口，如 https://localhost:7390
```

终端2 - 启动前端（需先配置后端地址）：

编辑`AgentGroupChat.Web/wwwroot/appsettings.json`：

```
{
  "AgentHostUrl": "https://localhost:7390"
}
```

然后启动：

```
cd agent-groupchat/AgentGroupChat.Web
dotnet run
```

访问前端地址（如`https://localhost:5001`）

### 方式二：使用DeepSeek

![img](https://img2024.cnblogs.com/blog/1690009/202510/1690009-20251028222255385-1659064413.png)

1. **获取API密钥**

访问 [DeepSeek开放平台](https://github.com)

2. **配置**

```
{
  "DefaultModelProvider": "OpenAI",
  "OpenAI": {
    "BaseUrl": "https://api.deepseek.com/v1",
    "ModelName": "deepseek-chat",
    "ApiKey": "sk-your-api-key"
  }
}
```

### 方式三：使用Azure AI Foundry

![img](https://img2024.cnblogs.com/blog/1690009/202510/1690009-20251028222318259-1078875047.png)

1. **创建Azure OpenAI资源**

访问 [AI Foundry](https://github.com)，创建Azure OpenAI服务。

2. **部署模型**

在AI Foundry中部署 `gpt-4o-mini` 模型。

3. **配置**

```
{
  "DefaultModelProvider": "AzureOpenAI",
  "AzureOpenAI": {
    "Endpoint": "https://your-resource.openai.azure.com/",
    "DeploymentName": "gpt-4o-mini",
    "ApiKey": "your-api-key"
  }
}
```

## 实战演示

### 测试场景1：直接提及

**输入**：`@莉子 推荐几部最近的热门动漫`

**路由过程**：

1. Triage Agent检测到 `@莉子`
2. 调用 `handoff_to_rina` 函数
3. 莉子接收消息并回复

**输出**：

```
莉子：哇！最近的新番超棒的呢！✨
强烈推荐《葬送的芙莉莲》，这部番真的是神作级别！
还有《咒术回战》第二季也超燃的！
如果喜欢轻松搞笑的，《关于我转生变成史莱姆这档事》第三季也很有趣哦~ (◕‿◕✿)
```

### 测试场景2：话题路由

**输入**：`最近在学习机器学习，有什么建议吗？`

**路由过程**：

1. Triage Agent识别"机器学习"属于科技话题
2. 路由到克洛伊（科技专家）

**输出**：

```
克洛伊：机器学习入门的话，建议从这几个方面开始：

1. **数学基础**：线性代数、概率统计、微积分
2. **编程**：Python是首选，熟练使用NumPy、Pandas
3. **经典算法**：先理解监督学习（线性回归、决策树）
4. **框架**：PyTorch或TensorFlow二选一
5. **实战项目**：Kaggle上有很多适合新手的数据集

推荐课程：吴恩达的Machine Learning课程（Coursera）
```

### 测试场景3：上下文连贯

**对话1**：

```
用户：什么是存在主义？
艾莲：存在主义认为"存在先于本质"，强调个体的自由选择和责任...
```

**对话2**（紧接上文）：

```
用户：那萨特的观点具体是什么？
系统：检测到话题延续，继续路由到艾莲
艾莲：萨特是存在主义的代表人物，他在《存在与虚无》中提出...
```

## 核心代码详解

### 1. WorkflowManager核心逻辑

```
public class WorkflowManager
{
    private Workflow BuildHandoffWorkflow(AgentGroup group)
    {
        // 1. 创建Triage Agent
        var triageAgent = new ChatClientAgent(
            _chatClient,
            group.TriageSystemPrompt ?? DefaultTriagePrompt,
            "triage",
            "Routes messages to appropriate agent"
        );
        
        // 2. 加载组内所有智能体
        var specialists = group.AgentIds
            .Select(id => _agentRepository.Get(id))
            .Where(a => a != null)
            .Select(a => new ChatClientAgent(
                _chatClient,
                a.SystemPrompt,
                a.Id,
                a.Description
            ))
            .ToList();
        
        // 3. 构建Handoff工作流
        var builder = AgentWorkflowBuilder
            .CreateHandoffBuilderWith(triageAgent)
            .WithHandoffs(triageAgent, specialists)  // Triage可以切换到任何专家
            .WithHandoffs(specialists, triageAgent); // 专家可以切回Triage
        
        return builder.Build();
    }
}
```

### 2. 事件流处理

```
public async Task> SendMessageAsync(
    string message, 
    string sessionId, 
    string? groupId = null)
{
    var summaries = new List();
    
    // 添加用户消息
    summaries.Add(new ChatMessageSummary
    {
        Content = message,
        IsUser = true,
        Timestamp = DateTime.UtcNow
    });
    
    // 获取工作流
    var workflow = _workflowManager.GetOrCreateWorkflow(groupId);
    
    // 加载历史消息
    var messages = LoadHistoryMessages(sessionId);
    messages.Add(new AIChatMessage(ChatRole.User, message));
    
    // 运行工作流
    await using StreamingRun run = await InProcessExecution.StreamAsync(workflow, messages);
    await run.TrySendMessageAsync(new TurnToken(emitEvents: true));
    
    string? currentExecutorId = null;
    ChatMessageSummary? currentSummary = null;
    
    // 处理事件流
    await foreach (WorkflowEvent evt in run.WatchStreamAsync())
    {
        if (evt is AgentRunUpdateEvent agentUpdate)
        {
            // 跳过Triage Agent的输出（它只负责路由）
            var executorIdPrefix = agentUpdate.ExecutorId.Split('_')[0];
            if (executorIdPrefix.Equals("triage", StringComparison.OrdinalIgnoreCase))
            {
                continue;
            }
            
            // 检测到新的specialist agent
            if (agentUpdate.ExecutorId != currentExecutorId)
            {
                currentExecutorId = agentUpdate.ExecutorId;
                var profile = _agentRepository.Get(currentExecutorId);
                
                currentSummary = new ChatMessageSummary
                {
                    AgentId = currentExecutorId,
                    AgentName = profile?.Name ?? currentExecutorId,
                    AgentAvatar = profile?.Avatar ?? "🤖",
                    Content = "",
                    IsUser = false,
                    Timestamp = DateTime.UtcNow
                };
                
                summaries.Add(currentSummary);
            }
            
            // 累积文本内容
            if (currentSummary != null)
            {
                var textContent = agentUpdate.Update.Contents
                    .OfType()
                    .FirstOrDefault();
                
                if (textContent != null && !string.IsNullOrWhiteSpace(textContent.Text))
                {
                    currentSummary.Content += textContent.Text;
                }
            }
        }
    }
    
    // 保存到数据库
    SaveMessages(sessionId, summaries);
    
    return summaries.Where(s => !s.IsUser).ToList();
}
```

### 3. Blazor WebAssembly前端

**为什么选择Blazor WASM？**

* 完全在浏览器运行,无需服务器端SignalR连接
* 与后端API完全解耦,便于扩展
* 利用.NET生态,C#编写前端逻辑

**API客户端封装**（`AgentHostClient.cs`）：

```
public class AgentHostClient
{
    private readonly HttpClient _httpClient;

    public AgentHostClient(HttpClient httpClient, ILogger logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    // 发送消息
    public async Task> SendMessageAsync(
        string sessionId, string message, string? groupId = null)
    {
        var request = new { Message = message, SessionId = sessionId, GroupId = groupId };
        var response = await _httpClient.PostAsJsonAsync("api/chat", request);
        return await response.Content.ReadFromJsonAsync>() ?? [];
    }

    // 获取智能体列表
    public async Task<List<AgentProfile>> GetAgentsAsync()
    {
        return await _httpClient.GetFromJsonAsync>("api/agents") ?? [];
    }
}
```

**主页面交互**（`Home.razor`）：

```
@page "/"
@inject AgentHostClient AgentHostClient
@inject IJSRuntime JSRuntime

<MudContainer MaxWidth="MaxWidth.False">
    <MudPaper Elevation="2" Class="chat-container">
        
        <div id="messages-container" class="messages-area">
            @foreach (var msg in _messages)
            {
                @if (msg.IsUser)
                {
                    <div class="user-message">@msg.Contentdiv>
                }
                else
                {
                    <div class="agent-message">
                        <MudAvatar>@msg.AgentAvatarMudAvatar>
                        <div>
                            <strong>@msg.AgentNamestrong>
                            <div>@((MarkupString)Markdown.ToHtml(msg.Content))div>
                        div>
                    div>
                }
            }
        div>

        
        <MudTextField @bind-Value="_inputMessage" 
                      Placeholder="输入消息..." 
                      OnKeyDown="HandleKeyPress" />
        <MudButton OnClick="SendMessage" Disabled="_isSending">发送MudButton>
    MudPaper>
MudContainer>

@code {
    private string _inputMessage = "";
    private List<ChatMessageSummary> _messages = new();
    private bool _isSending = false;

    private async Task SendMessage()
    {
        if (string.IsNullOrWhiteSpace(_inputMessage)) return;
        
        _isSending = true;
        try
        {
            // 调用后端API
            var response = await AgentHostClient.SendMessageAsync(
                _currentSession.Id, 
                _inputMessage,
                _currentSession.GroupId
            );
            
            // 更新消息列表
            _messages.AddRange(response);
            
            // 自动滚动到底部
            await JSRuntime.InvokeVoidAsync("smoothScrollToBottom", "messages-container");
        }
        finally
        {
            _isSending = false;
            _inputMessage = "";
        }
    }

    // Enter键发送,Shift+Enter换行
    private async Task HandleKeyPress(KeyboardEventArgs e)
    {
        if (e.Key == "Enter" && !e.ShiftKey)
        {
            await SendMessage();
        }
    }
}
```

**MudBlazor组件库**：

项目使用MudBlazor构建现代化UI：

```
<MudPaper Elevation="2" Class="pa-4">
    <MudText Typo="Typo.h5">标题MudText>
MudPaper>


<MudTextField @bind-Value="value" Label="标签" Variant="Variant.Outlined" />


<MudButton Variant="Variant.Filled" Color="Color.Primary">按钮MudButton>


<MudAvatar Color="Color.Primary">🧠MudAvatar>
```

## 总结与展望

通过这个AI美女聊天群组项目，我们学习了：

1. **大模型基础**：理解LLM API、Function Calling和MCP协议
2. **多智能体架构**：掌握Handoff模式和Triage智能路由
3. **Agent Framework**：使用`AgentWorkflowBuilder`构建Handoff工作流
4. **Aspire编排**：通过.NET Aspire实现服务发现和统一启动
5. **Blazor WASM**：前后端分离架构，完全客户端渲染
6. **实战技巧**：动态加载、状态管理、LiteDB持久化

### 参考资源

**官方文档**

* [Microsoft Agent Framework 文档](https://github.com):[闪电加速器](https://sanzhijia.com)
* [Agent Framework GitHub](https://github.com)
* [Azure AI Foundry](https://github.com)

**国内平台**

* [阿里云百炼](https://github.com)
* [DeepSeek API](https://github.com)

**示例代码**

* [agent-groupchat 完整代码](https://github.com)

---

*如果这篇文章对你有帮助，欢迎点赞收藏！有任何问题也欢迎在评论区讨论。*
