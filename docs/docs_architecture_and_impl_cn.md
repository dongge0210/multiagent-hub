# 多智能体C++系统结构与实现框架说明

## 一、项目目录结构

```
/your_project_root/
├── CMakeLists.txt                     # 构建或VS工程文件
├── README.md
├── config/
│   ├── api_config.json                # LLM API、模型、并发、Token等配置
│   ├── permissions.json               # 权限/审批/串门规则配置
│   └── agent_matrix.json              # agent间互访关系等
├── src/
│   ├── cli/
│   │   └── main.cpp                   # 命令行主入口
│   │   └── CliCommandParser.h/.cpp
│   ├── mas/
│   │   ├── MASManager.h/.cpp          # 多Agent中央调度/权限/审批/组队/日志（MAS）
│   ├── agents/
│   │   ├── IAgent.h
│   │   ├── LeaderAgent.h/.cpp
│   │   ├── MidAgent.h/.cpp
│   │   ├── WorkerAgent.h/.cpp
│   │   └── SubAgentRegistry.h/.cpp    # agent发现与组队
│   ├── llm/
│   │   ├── LLMAPIManager.h/.cpp       # LLM API管理与调用
│   │   ├── ModelSelector.h/.cpp       # 模型选择与路由
│   │   └── TokenManager.h/.cpp        # Token使用与限制管理
│   ├── mcp/
│   │   ├── MCPClient.h/.cpp           # MCP客户端，AI调用工具的接口
│   │   ├── ToolRegistry.h/.cpp        # 工具注册与管理
│   │   └── ToolInvoker.h/.cpp         # 工具调用执行器
│   ├── tools/
│   │   ├── ITool.h
│   │   ├── FileTool.h/.cpp
│   │   ├── DBTool.h/.cpp
│   │   └── ...
│   ├── db/
│   │   ├── DataManager.h/.cpp
│   │   ├── ApprovalQueue.h/.cpp
│   │   └── LogManager.h/.cpp
│   ├── common/
│   │   ├── Task.h/.cpp
│   │   └── Utility.h/.cpp
├── test/
│   └── test_scenarios.cpp
└── doc/
    ├── system_flow.png
    └── agent_collaboration.md
```

---

## 二、整体架构/流程

- 所有CLI命令，优先交给MASManager调度。
- MASManager负责权限、Agent调度、任务分配、审批流、组队和日志（整体系统称为 MAS）。
- Agent分为三层：Leader/Mid/Worker，均通过注册中心（SubAgentRegistry）动态发现、协同、串门。
- 所有Agent通过MCP（Model Context Protocol）调用工具，MCP是AI调用工具的标准协议接口。

注：文中 "MCP" 指 "模型上下文协议（Model Context Protocol）"，它是AI调用工具的标准协议，提供统一的工具调用接口。MCP客户端负责将Agent的请求转换为具体的工具调用，管理工具注册和执行。

## 三、实现要点与机制

### 1. MAS（中央调度与整体）

- `MASManager`负责 agent 注册、权限校验、任务路由、审批流和组队。MAS 作为整体负责多智能体协作与资源调度。
- MAS不直接管理工具调用，而是通过MCP客户端与工具系统交互。

### 2. Agent三层分工与注册
- 所有agent必须实现`IAgent`接口，支持标准的“接收任务-调用工具-交互-回传结果”。
- LeaderAgent负责最后审批写入、全局分派；MidAgent可执行或串接任务但核心写入需审批；WorkerAgent执行具体工具调用。
- 所有agent通过`SubAgentRegistry`注册和发现，支持临时组队（自由协作），实现agent间“串门”。
- 权限、可见/可协作关系由`agent_matrix.json`配置。

### 3. LLM API管理
- `LLMAPIManager`负责管理所有网络LLM API的调用，包括OpenAI、Claude、本地模型等。
- `ModelSelector`根据任务类型和复杂度动态选择最适合的LLM模型。
- `TokenManager`监控和控制Token使用量，防止超出配额限制。

### 4. MCP工具调用系统
- `MCPClient`提供统一的工具调用接口，所有Agent通过MCP协议调用工具。
- `ToolRegistry`负责工具的注册、发现和管理，支持热插拔。
- `ToolInvoker`负责执行具体的工具调用，处理参数传递和结果返回。
- 所有工具实现`ITool`接口，通过MCP协议暴露给AI模型调用。

### 5. 数据审批与日志
- `DataManager`与`ApprovalQueue`组合管理正式和临时写入，所有需审批的数据先入排队，获得Leader批复后正式持久化。
- `LogManager`全链路记录任务、审批、Agent协作（debug和回溯用）。

### 6. 配置与热扩展
- 全部行为规则、权限关系、模型API、插件启用等，均通过config下json进行集中维护，即改即用无需重编译。
- 插件拓展能力和Agent注册也可支持动态(进阶可上DLL、so方案或直接源码静态链接）。

---

## 四、开发和部署建议

1. **目录搭起后，逐步落每个目录的基类/核心流程接口**，如IAgent、ITool等。
2. **先实现MAS、Agent、Tools之间调用主流程**，可用模拟数据或mock工具实现通过“命令-路由-审批-写入-日志”的完整最小链路（能跑起来为准）。
3. **逐步替换Mock为真实API和工具实现**，并增加测试覆盖常见串门、组队、超限等场景。
4. 测试建议直接写在test/目录下，细分任务流、审批边界、多agent协作等用例。

---

## 五、常见开发注意事项

- 并发管理务必用互斥锁、条件变量防数据竞争。
- 所有任务与审批请求建议用唯一ID（可用UUID库）。
- 插件API、配置格式定期review；建议写接口注释，以方便扩展者参考。
- 推荐用VS2022/VS Code+cmake，文件与依赖见CMakeLists.txt，严格按src/目录组织源码，便于多人协作。
- 日志与出错处理先保证信息全，后期再做细粒度异常和自动修复。
- 需求变化/新插件建议写清文档，在doc/目录补结构说明/变更集。

---

## 六、FAQ

- **如何加新agent或工具？**  
  实现IAgent/ITool接口后，在SubAgentRegistry/DataManager等地注册，配置agent_matrix.json分配权限即可。
- **LLM API速率或额度怎么配？**  
  直接改config/api_config.json，自定义最大并发、token池等。LLMAPIManager负责统一管理各种LLM API的调用和限制。

---

## 工作流程（Mermaid）

下面是系统典型工作流程图（Mermaid 格式）。把这段放到文档里可以在支持 mermaid 的渲染器里直接看到流程图。

```mermaid
flowchart TD
    CLI[CLI 用户输入] --> MASManager

    subgraph 核心系统
        MASManager[MASManager]
        SubAgentRegistry[SubAgentRegistry]
        ApprovalQueue[ApprovalQueue]
        DataManager[DataManager]
        LogManager[LogManager]

        MASManager --> SubAgentRegistry
        MASManager --> ApprovalQueue
        MASManager --> DataManager
        MASManager --> LogManager

        ApprovalQueue --> LogManager
        MASManager --> LogManager
    end

    subgraph Agent层
        LeaderAgent[LeaderAgent]
        MidAgent[MidAgent]
        WorkerAgent[WorkerAgent]

        SubAgentRegistry --> LeaderAgent
        LeaderAgent --> MidAgent
        MidAgent --> WorkerAgent

        %% Worker和系统的连接
        WorkerAgent --> LLMAPIManager
        WorkerAgent --> MCPClient
        WorkerAgent --> ApprovalQueue
        WorkerAgent --> LogManager
    end

    subgraph LLM系统
        LLMAPIManager[LLMAPIManager]
        ModelSelector[ModelSelector]
        TokenManager[TokenManager]

        LLMAPIManager --> ModelSelector
        LLMAPIManager --> TokenManager

        %% LLMAPI与外部服务连接
        LLMAPIManager --> OpenAI
        LLMAPIManager --> Claude
        LLMAPIManager --> LocalLLM

        LLMAPIManager --> WorkerAgent
    end

    subgraph MCP系统
        MCPClient[MCPClient]
        ToolRegistry[ToolRegistry]
        ToolInvoker[ToolInvoker]

        MCPClient --> ToolRegistry
        ToolRegistry --> ToolInvoker
        ToolInvoker --> FileTool
        ToolInvoker --> DBTool
        ToolInvoker --> APITool
        ToolInvoker --> MCPClient
        MCPClient --> WorkerAgent
    end

    subgraph 工具层
        FileTool[文件操作工具]
        DBTool[数据库工具]
        APITool[API调用工具]

        FileTool --> ToolInvoker
        DBTool --> ToolInvoker
        APITool --> ToolInvoker
    end

    subgraph 外部服务
        OpenAI[OpenAI API]
        Claude[Claude API]
        LocalLLM[本地LLM服务]
    end

    %% 审批、持久化
    ApprovalQueue --> LeaderAgent
    LeaderAgent --> DataManager
    DataManager --> DBTool
```

---
