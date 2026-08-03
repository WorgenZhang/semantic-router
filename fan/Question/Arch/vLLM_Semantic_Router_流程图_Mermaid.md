# vLLM Semantic Router 架构流程图（Mermaid）

> 本文档使用 Mermaid 语法绘制完整的架构流程图，包括决策流程、子模块处理逻辑、代码调用时序图和关键代码位置。

---

## 1. 系统整体架构图

```mermaid
graph TB
    subgraph Datasets["📊 Datasets"]
        MMLU["MMLU-Pro<br/>Dataset"]
        JB["Jailbreak<br/>Datasets"]
        MSP["Microsoft Presidio<br/>Dataset"]
        CD["Customer<br/>Dataset"]
        PCD["Private Customer<br/>Dataset"]
    end

    subgraph Training["🔧 Fine-tuning ModernBERT"]
        FT["Fine-tuning ModernBERT<br/>for Intent Classification"]
        FT_LORA1["LoRA Classifier<br/>Fine-tuning"]
        FT_LORA2["LoRA PromptGuard<br/>Fine-tuning"]
        FT_PII["LoRA PII<br/>Fine-tuning"]
    end

    subgraph VLLMRouter["🎯 vLLM Semantic Router"]
        subgraph EnvoyLayer["Cloud Native Layer"]
            EP["Cloud Native High Performance<br/>EnvoyProxy"]
            GRPC["Golang with Rust Binding<br/>for Envoy ExtProc Interface"]
        end

        subgraph CoreEngine["Core Classification Engine"]
            BM["Small Base Model<br/>for Intent Classification<br/>(ModernBERT)"]
            LA1["Enhanced LoRA<br/>Adapter 1"]
            LA2["Enhanced LoRA<br/>Adapter 2"]
            CB["Continuous<br/>Batching"]
            RC["Rust Core for<br/>High-Performance<br/>Classification"]
        end

        subgraph Dependencies["Dependencies"]
            CF["Candle<br/>Framework"]
            HF["HuggingFace<br/>Tokenizers Library"]
        end
    end

    %% Dataset connections
    MMLU -.-> FT
    JB -.-> FT_LORA2
    MSP -.-> FT_PII
    CD -.-> FT
    PCD -.-> FT

    %% Training to Model
    FT ==> BM
    FT_LORA1 ==> LA1
    FT_LORA2 ==> LA2

    %% Core flow
    EP --> GRPC
    GRPC --> RC
    RC --> BM
    BM --> LA1
    BM --> LA2
    LA1 --> CB
    LA2 --> CB
    CB --> RC

    %% Dependencies
    CF --> RC
    HF --> RC

    classDef dataset fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    classDef training fill:#FCE4EC,stroke:#C62828,color:#B71C1C
    classDef envoy fill:#FFF9C4,stroke:#F9A825,color:#F57F17
    classDef core fill:#E3F2FD,stroke:#1565C0,color:#0D47A1
    classDef dep fill:#F3E5F5,stroke:#7B1FA2,color:#4A148C

    class MMLU,JB,MSP,CD,PCD dataset
    class FT,FT_LORA1,FT_LORA2,FT_PII training
    class EP,GRPC envoy
    class BM,LA1,LA2,CB,RC core
    class CF,HF dep
```

---

## 2. 完整请求处理决策流程

```mermaid
flowchart TD
    Start(["🚀 用户请求到达"])
    Start --> Envoy["EnvoyProxy 接收请求"]
    Envoy --> ExtProc["ExtProc gRPC 调用<br/>OpenAIRouter"]
    ExtProc --> ReqHeader["处理请求头<br/>#60;processor_req_header.go#62;"]
    ReqHeader --> ReqBody["处理请求体<br/>#60;processor_req_body.go#62;"]

    ReqBody --> FilterChain{"📋 请求过滤器链"}

    FilterChain --> F1["① Classification 意图分类<br/>#60;req_filter_classification.go#62;"]
    F1 --> F2["② Jailbreak 越狱检测<br/>#60;req_filter_jailbreak.go#62;"]
    F2 --> F2_Check{"越狱检测结果?"}
    F2_Check -->|"检测到越狱"| Block["🚫 拦截请求<br/>返回安全响应"]
    F2_Check -->|"安全"| F3["③ PII 个人信息检测<br/>#60;req_filter_pii.go#62;"]
    F3 --> F3_Check{"检测到 PII?"}
    F3_Check -->|"是"| Mask["🔒 PII 脱敏处理"]
    F3_Check -->|"否"| F4
    Mask --> F4["④ Cache 语义缓存检查<br/>#60;req_filter_cache.go#62;"]
    F4 --> F4_Check{"缓存命中?"}
    F4_Check -->|"命中"| CacheHit["⚡ 直接返回缓存结果"]
    F4_Check -->|"未命中"| F5["⑤ Reasoning 推理决策<br/>#60;req_filter_reason.go#62;"]
    F5 --> F6["⑥ RAG 检索增强<br/>#60;req_filter_rag.go#62;"]
    F6 --> F7["⑦ System Prompt 注入<br/>#60;req_filter_sys_prompt.go#62;"]
    F7 --> F8["⑧ Tools 工具调用路由<br/>#60;req_filter_tools.go#62;"]

    F8 --> Decision["Decision Engine 决策引擎<br/>#60;decision/engine.go#62;"]
    Decision --> Routing{"🔀 路由策略?"}

    Routing -->|"Auto"| Auto["handleAutoModelRouting<br/>自动路由选择"]
    Routing -->|"Specified"| Spec["handleSpecifiedModelRouting<br/>指定模型路由"]
    Routing -->|"Anthropic"| Anthro["handleAnthropicRouting<br/>Claude API 路由"]

    Auto --> SelectModel["选择最优模型+端点<br/>selectEndpointForModel"]
    Spec --> SelectModel
    Anthro --> TransformAPI["OpenAI → Anthropic<br/>格式转换"]
    TransformAPI --> SelectModel

    SelectModel --> SendToLLM["📤 路由到 LLM Backend"]

    SendToLLM --> ResHeader["处理响应头<br/>#60;processor_res_header.go#62;"]
    ResHeader --> ResBody["处理响应体<br/>#60;processor_res_body.go#62;"]
    ResBody --> Hallucination{"幻觉检测?"}
    Hallucination -->|"检测到幻觉"| HalluFlag["⚠️ 标记幻觉"]
    Hallucination -->|"正常"| Return
    HalluFlag --> Return(["✅ 返回响应给用户"])

    style Start fill:#4CAF50,color:#fff
    style Block fill:#F44336,color:#fff
    style CacheHit fill:#FF9800,color:#fff
    style Return fill:#4CAF50,color:#fff
    style Decision fill:#9C27B0,color:#fff
```

---

## 3. Rust 核心分类引擎处理逻辑

```mermaid
flowchart TD
    Input(["文本输入"])
    Input --> Tokenize["HuggingFace Tokenizers<br/>文本分词"]
    Tokenize --> Unified{"统一分类器<br/>UnifiedClassifier"}

    Unified --> PathSelect{"DualPathRouter<br/>路径选择策略"}

    PathSelect -->|"Traditional Path"| Trad["传统 ModernBERT<br/>完整模型推理"]
    PathSelect -->|"LoRA Path"| LoRA["Base Model + LoRA Adapter<br/>融合推理"]

    subgraph TraditionalPath["Traditional Path"]
        Trad --> T_Embed["Embedding Layer"]
        T_Embed --> T_Encoder["Transformer Encoder"]
        T_Encoder --> T_Head["Classification Head"]
        T_Head --> T_Result["分类结果"]
    end

    subgraph LoRAPath["LoRA Path"]
        LoRA --> L_Base["Base Model Forward"]
        L_Base --> L_LoRA["LoRA Adapter Merge<br/>(W + α·B·A)"]
        L_LoRA --> L_Head["Classification Head"]
        L_Head --> L_Result["分类结果"]
    end

    T_Result --> Batch["Continuous Batching<br/>批处理聚合"]
    L_Result --> Batch

    Batch --> Output(["输出: Category + Confidence"])

    style Input fill:#E3F2FD,color:#0D47A1
    style Output fill:#E8F5E9,color:#1B5E20
    style PathSelect fill:#FFF3E0,color:#E65100
```

---

## 4. DualPathRouter 智能路由决策流程

```mermaid
flowchart TD
    Req(["处理需求<br/>ProcessingRequirements"])
    Req --> Strategy{"PathSelectionStrategy<br/>选择策略?"}

    Strategy -->|"PreferTraditional"| UseTrad["使用 Traditional Path"]
    Strategy -->|"PreferLoRA"| UseLoRA["使用 LoRA Path"]
    Strategy -->|"Automatic"| AutoSelect

    subgraph AutoSelect["自动选择逻辑"]
        AS1{"需要 LoRA 特定任务?<br/>(jailbreak/pii/mmlu)"}
        AS1 -->|"是"| AS_LoRA["选择 LoRA Path"]
        AS1 -->|"否"| AS2{"需要最低延迟?"}
        AS2 -->|"是"| AS_Trad["选择 Traditional Path"]
        AS2 -->|"否"| AS3{"需要最高精度?"}
        AS3 -->|"是"| AS_Check["检查 LoRA 适配器可用性"]
        AS3 -->|"否"| AS_Default["默认: Traditional"]

        AS_Check -->|"可用"| AS_LoRA2["选择 LoRA Path"]
        AS_Check -->|"不可用"| AS_Trad2["选择 Traditional Path"]
    end

    Strategy -->|"PerformanceBased"| PerfSelect

    subgraph PerfSelect["基于性能的选择"]
        PS1["获取 Traditional 历史性能"]
        PS2["获取 LoRA 历史性能"]
        PS1 --> PS3["计算 Traditional 得分<br/>calculate_path_score"]
        PS2 --> PS4["计算 LoRA 得分<br/>calculate_path_score"]
        PS3 --> PS5{"比较得分"}
        PS4 --> PS5
        PS5 -->|"Traditional 更高"| PS_T["选择 Traditional"]
        PS5 -->|"LoRA 更高"| PS_L["选择 LoRA"]
    end

    Strategy -->|"Intelligent"| IntelSelect

    subgraph IntelSelect["超级智能选择"]
        IS1["分析任务类型"]
        IS2["评估批大小"]
        IS3["检查历史性能"]
        IS4["考虑精度要求"]
        IS5["评估延迟约束"]
        IS1 --> IS6["综合评分"]
        IS2 --> IS6
        IS3 --> IS6
        IS4 --> IS6
        IS5 --> IS6
        IS6 --> IS7{"选择最优路径"}
    end

    UseTrad --> Result(["PathSelection<br/>路径 + 置信度 + 原因"])
    UseLoRA --> Result
    AS_LoRA --> Result
    AS_Trad --> Result
    AS_Default --> Result
    AS_LoRA2 --> Result
    AS_Trad2 --> Result
    PS_T --> Result
    PS_L --> Result
    IS7 --> Result

    style Req fill:#E3F2FD,color:#0D47A1
    style Result fill:#E8F5E9,color:#1B5E20
    style Strategy fill:#FFF3E0,color:#E65100
```

---

## 5. Go 分类引擎调用时序图

```mermaid
sequenceDiagram
    participant User as 用户请求
    participant Envoy as EnvoyProxy
    participant ExtProc as ExtProc Server<br/>(server.go)
    participant Router as OpenAIRouter<br/>(router.go)
    participant Body as RequestBodyProcessor<br/>(processor_req_body.go)
    participant Filter as FilterChain<br/>(req_filter_*.go)
    participant Classifier as Classifier<br/>(classifier.go)
    participant Candle as Candle Binding<br/>(semantic-router.go)
    participant Rust as Rust Core<br/>(candle-binding/src/)
    participant Decision as Decision Engine<br/>(engine.go)
    participant Backend as LLM Backend

    User->>Envoy: HTTP/gRPC 请求
    Envoy->>ExtProc: ExtProc gRPC call

    Note over ExtProc,Router: 阶段1: 请求头处理
    ExtProc->>Router: Process(RequestHeaders)
    Router->>Router: handleRequestHeaders()

    Note over ExtProc,Router: 阶段2: 请求体处理
    ExtProc->>Router: Process(RequestBody)
    Router->>Body: handleRequestBody()

    Note over Body,Filter: 阶段3: 过滤器链执行
    Body->>Filter: Classification Filter
    Filter->>Classifier: Classify(text)

    Note over Classifier,Rust: 阶段4: Rust 分类引擎调用
    Classifier->>Candle: CategoryInference.Classify()
    Candle->>Rust: FFI call (C binding)
    Rust->>Rust: Tokenize (HF Tokenizers)
    Rust->>Rust: DualPathRouter.select_path()

    alt Traditional Path
        Rust->>Rust: ModernBERT forward()
    else LoRA Path
        Rust->>Rust: Base + LoRA merge forward()
    end

    Rust->>Rust: Continuous Batching
    Rust-->>Candle: ClassResult {label, confidence}
    Candle-->>Classifier: 分类结果

    Note over Filter,Classifier: 阶段5: 安全过滤
    Filter->>Classifier: Jailbreak Detection
    Classifier->>Candle: JailbreakInference.Classify()
    Candle-->>Classifier: 是否越狱

    Filter->>Classifier: PII Detection
    Classifier->>Candle: PIIInference.Classify()
    Candle-->>Classifier: PII entities

    Note over Body,Decision: 阶段6: 决策与路由
    Body->>Decision: MakeDecision(signals)
    Decision-->>Body: DecisionResult {model, endpoint}

    Body->>Router: Routing Response

    alt Auto Routing
        Router->>Router: handleAutoModelRouting()
    else Specified Model
        Router->>Router: handleSpecifiedModelRouting()
    else Anthropic
        Router->>Router: handleAnthropicRouting()
    end

    Router->>Router: selectEndpointForModel()
    Router-->>Envoy: ProcessingResponse (headers + body mutations)
    Envoy->>Backend: 路由到目标 LLM

    Note over Envoy,Backend: 阶段7: 响应处理
    Backend-->>Envoy: LLM 响应
    Envoy->>Router: Process(ResponseHeaders)
    Envoy->>Router: Process(ResponseBody)
    Router->>Classifier: Hallucination Detection
    Router-->>Envoy: ProcessingResponse
    Envoy-->>User: 最终响应
```

---

## 6. LoRA 训练与推理流程

```mermaid
flowchart LR
    subgraph TrainingPhase["🔧 训练阶段 (Python)"]
        D1["MMLU-Pro Dataset"]
        D2["Jailbreak Dataset"]
        D3["Presidio Dataset"]
        D4["Customer Dataset"]

        D1 --> T1["classifier_model_fine_tuning_lora/"]
        D2 --> T2["prompt_guard_fine_tuning_lora/"]
        D3 --> T3["pii_model_fine_tuning_lora/"]
        D4 --> T1

        T1 --> M1["LoRA Adapter weights<br/>(lora_A, lora_B, alpha)"]
        T2 --> M2["LoRA PromptGuard weights"]
        T3 --> M3["LoRA PII weights"]
    end

    subgraph InferencePhase["⚡ 推理阶段 (Rust)"]
        Load["model_factory.rs<br/>ModelFactory::create()"]
        
        M1 --> Load
        M2 --> Load
        M3 --> Load

        Load --> Adapter["lora_adapter.rs<br/>LoraAdapter::load()"]
        Adapter --> Merge["bert_lora.rs<br/>BertWithLoRA::forward()"]

        Base["Base ModernBERT<br/>pretrained weights"] --> Merge

        Merge --> Result["Enhanced Classification<br/>Result"]
    end

    style TrainingPhase fill:#FFF3E0
    style InferencePhase fill:#E3F2FD
```

---

## 7. Envoy ExtProc 交互流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Envoy as Envoy Proxy<br/>(envoy.yaml)
    participant ExtProc as ExtProc Server<br/>(Go Service)

    Client->>Envoy: POST /v1/chat/completions
    
    Note over Envoy,ExtProc: Phase 1: Request Headers
    Envoy->>ExtProc: ProcessingRequest{RequestHeaders}
    ExtProc->>ExtProc: 提取 path, method, content-type
    ExtProc-->>Envoy: ProcessingResponse{HeaderMutation}

    Note over Envoy,ExtProc: Phase 2: Request Body
    Envoy->>ExtProc: ProcessingRequest{RequestBody}
    ExtProc->>ExtProc: 解析 OpenAI 请求格式
    ExtProc->>ExtProc: 执行过滤器链
    ExtProc->>ExtProc: 分类 + 决策 + 路由
    ExtProc-->>Envoy: ProcessingResponse{<br/>HeaderMutation: route_to_cluster,<br/>BodyMutation: modified_body<br/>}

    Note over Envoy: Envoy 根据 header mutation<br/>路由到对应后端集群
    Envoy->>Envoy: Route to target cluster

    Note over Envoy,ExtProc: Phase 3: Response Headers
    Envoy->>ExtProc: ProcessingRequest{ResponseHeaders}
    ExtProc->>ExtProc: 启动上游请求 span
    ExtProc-->>Envoy: ProcessingResponse

    Note over Envoy,ExtProc: Phase 4: Response Body
    Envoy->>ExtProc: ProcessingRequest{ResponseBody}
    ExtProc->>ExtProc: 幻觉检测
    ExtProc->>ExtProc: PII 脱敏(响应)
    ExtProc-->>Envoy: ProcessingResponse{BodyMutation}

    Envoy-->>Client: HTTP Response
```

---

## 8. 分类引擎初始化流程

```mermaid
flowchart TD
    Start(["服务启动"])
    Start --> NewRouter["NewOpenAIRouter(configPath)<br/>router.go:45"]

    NewRouter --> LoadConfig["加载配置<br/>config.yaml"]
    LoadConfig --> InitClassifier["初始化 Classifier<br/>classifier.go"]

    InitClassifier --> InitCat{"Category 分类器<br/>CategoryInitializerImpl.Init()"}
    InitCat --> CatModel["加载 ModernBERT<br/>Intent 分类模型"]

    InitClassifier --> InitJB{"Jailbreak 分类器<br/>JailbreakInitializerImpl.Init()"}
    InitJB --> JBCheck{"UseVLLM?"}
    JBCheck -->|"是"| JB_VLLM["使用外部 vLLM<br/>Guardrail 模型"]
    JBCheck -->|"否"| JB_Candle["使用 Candle 本地推理<br/>加载 PromptGuard 模型"]

    InitClassifier --> InitPII{"PII 分类器<br/>PIIInitializerImpl.Init()"}
    InitPII --> PIIModel["加载 PII 检测模型<br/>(Presidio/Token分类)"]

    InitClassifier --> InitEmbed{"Embedding 分类器<br/>EmbeddingClassifier.Init()"}
    InitEmbed --> EmbedModel["加载 Embedding 模型<br/>(语义相似度)"]

    InitClassifier --> InitHallu{"Hallucination 检测器<br/>HallucinationDetector.Init()"}
    InitHallu --> HalluModel["加载幻觉检测模型"]

    CatModel --> Ready
    JB_VLLM --> Ready
    JB_Candle --> Ready
    PIIModel --> Ready
    EmbedModel --> Ready
    HalluModel --> Ready

    Ready(["✅ 分类引擎就绪<br/>开始处理请求"])

    style Start fill:#4CAF50,color:#fff
    style Ready fill:#4CAF50,color:#fff
```

---

## 9. 关键代码位置索引

```mermaid
graph LR
    subgraph GoLayer["Go 服务层"]
        R1["extproc/router.go<br/>OpenAIRouter 主入口"]
        R2["extproc/server.go<br/>gRPC 服务器"]
        R3["extproc/processor_req_body.go<br/>请求体处理核心"]
        R4["extproc/req_filter_classification.go<br/>意图分类过滤器"]
        R5["classification/classifier.go<br/>分类器管理"]
        R6["classification/unified_classifier.go<br/>统一分类接口"]
        R7["decision/engine.go<br/>决策引擎"]
        R8["selection/<br/>模型选择策略"]
        R1 --> R2
        R2 --> R3
        R3 --> R4
        R4 --> R5
        R5 --> R6
        R3 --> R7
        R7 --> R8
    end

    subgraph RustLayer["Rust 核心引擎"]
        C1["candle-binding/semantic-router.go<br/>Go-Rust FFI 桥接"]
        C2["src/ffi/<br/>C FFI 接口定义"]
        C3["src/classifiers/unified.rs<br/>统一分类器实现"]
        C4["src/model_architectures/model_factory.rs<br/>模型工厂"]
        C5["src/model_architectures/routing.rs<br/>DualPathRouter 智能路由"]
        C6["src/model_architectures/lora/bert_lora.rs<br/>BERT+LoRA 融合推理"]
        C7["src/model_architectures/lora/lora_adapter.rs<br/>LoRA 适配器管理"]
        C1 --> C2
        C2 --> C3
        C3 --> C4
        C4 --> C5
        C5 --> C6
        C6 --> C7
    end

    subgraph TrainLayer["训练流水线"]
        T1["training/classifier_model_fine_tuning/<br/>基座模型微调"]
        T2["training/training_lora/<br/>LoRA 适配器训练"]
        T3["training/modernbert_dissat_pipeline/<br/>ModernBERT 情感分析"]
        T4["training/prompt_guard_fine_tuning/<br/>越狱检测训练"]
    end

    subgraph ConfigLayer["配置与部署"]
        CF1["config/config.yaml<br/>路由器主配置"]
        CF2["config/envoy.yaml<br/>Envoy 代理配置"]
        CF3["config/intelligent-routing/<br/>智能路由配置"]
        CF4["config/prompt-guard/<br/>安全防护配置"]
    end

    R6 --> C1
    T2 --> C7

    style GoLayer fill:#E3F2FD
    style RustLayer fill:#FFF3E0
    style TrainLayer fill:#FCE4EC
    style ConfigLayer fill:#E8F5E9
```

---

## 10. 端到端数据流状态图

```mermaid
stateDiagram-v2
    [*] --> RequestReceived: 用户请求到达

    RequestReceived --> HeaderProcessing: Envoy 拦截
    HeaderProcessing --> BodyProcessing: 请求头解析完成

    state BodyProcessing {
        [*] --> Parsing: 解析 OpenAI 格式
        Parsing --> Classification: 意图分类
        Classification --> JailbreakCheck: 安全检查

        JailbreakCheck --> PIICheck: 安全
        JailbreakCheck --> Blocked: 检测到越狱
        PIICheck --> CacheCheck: 通过
        PIICheck --> PIIMasking: 检测到 PII
        PIIMasking --> CacheCheck

        CacheCheck --> CacheHit: 缓存命中
        CacheCheck --> EnhancedProcessing: 缓存未命中

        state EnhancedProcessing {
            [*] --> ReasoningDecision
            ReasoningDecision --> RAGAugmentation
            RAGAugmentation --> ToolRouting
            ToolRouting --> [*]
        }

        EnhancedProcessing --> ModelSelection
        ModelSelection --> [*]
        CacheHit --> [*]
        Blocked --> [*]
    }

    BodyProcessing --> DecisionMaking: 过滤器链完成

    state DecisionMaking {
        [*] --> SignalAggregation: 信号聚合
        SignalAggregation --> RouteDecision: 决策计算
        RouteDecision --> EndpointSelection: 端点选择
        EndpointSelection --> [*]
    }

    DecisionMaking --> UpstreamRequest: 路由决策完成
    UpstreamRequest --> LLMProcessing: 发送到 LLM
    LLMProcessing --> ResponseProcessing: LLM 返回响应

    state ResponseProcessing {
        [*] --> HallucinationCheck: 幻觉检测
        HallucinationCheck --> ResponsePII: PII 审查
        ResponsePII --> FinalResponse: 最终处理
        FinalResponse --> [*]
    }

    ResponseProcessing --> ResponseDelivery: 响应处理完成
    ResponseDelivery --> [*]: 返回用户
```

---

## 附录: 关键文件路径速查表

| 模块 | 关键文件 | 说明 |
|------|----------|------|
| **Envoy 配置** | `config/envoy.yaml` | Envoy 代理配置 |
| **路由器主配置** | `config/config.yaml` | 路由策略、模型配置 |
| **ExtProc 入口** | `src/semantic-router/pkg/extproc/server.go` | gRPC 服务 |
| **请求路由** | `src/semantic-router/pkg/extproc/router.go` | `OpenAIRouter` |
| **请求体处理** | `src/semantic-router/pkg/extproc/processor_req_body.go` | 核心决策逻辑 |
| **分类过滤器** | `src/semantic-router/pkg/extproc/req_filter_classification.go` | 意图分类 |
| **越狱检测** | `src/semantic-router/pkg/extproc/req_filter_jailbreak.go` | 安全防护 |
| **PII 检测** | `src/semantic-router/pkg/extproc/req_filter_pii.go` | 隐私保护 |
| **缓存检查** | `src/semantic-router/pkg/extproc/req_filter_cache.go` | 语义缓存 |
| **推理决策** | `src/semantic-router/pkg/extproc/req_filter_reason.go` | 是否推理模型 |
| **分类器管理** | `src/semantic-router/pkg/classification/classifier.go` | 所有分类器 |
| **决策引擎** | `src/semantic-router/pkg/decision/engine.go` | 信号综合决策 |
| **Go-Rust 桥接** | `candle-binding/semantic-router.go` | FFI 绑定 |
| **Rust 统一分类** | `candle-binding/src/classifiers/unified.rs` | 统一分类器 |
| **模型工厂** | `candle-binding/src/model_architectures/model_factory.rs` | 模型创建 |
| **双路径路由** | `candle-binding/src/model_architectures/routing.rs` | `DualPathRouter` |
| **LoRA 融合** | `candle-binding/src/model_architectures/lora/bert_lora.rs` | BERT+LoRA |
| **LoRA 适配器** | `candle-binding/src/model_architectures/lora/lora_adapter.rs` | 适配器管理 |
| **模型训练** | `src/training/classifier_model_fine_tuning/` | 基座模型微调 |
| **LoRA 训练** | `src/training/training_lora/` | LoRA 适配器训练 |
