# RAGflow学习

# 一、项目核心定位

RAGFlow 是一个基于深度文档理解的开源 RAG（检索增强生成）引擎。它的核心价值主张是："深度文档理解 \+ 精准分块 \+ 混合检索"，而非简单的文本切分\+向量搜索。与 LangChain 等编排框架不同，RAGFlow 的差异化竞争力在于文档解析的质量——通过视觉模型\(YOLOv10\)做版面分析、通过 PaddleOCR 做文字识别、通过 14 种分块策略针对不同文档类型精准切割。

# 二、项目架构总览

```mermaid
flowchart TD
    subgraph L1["用户层"]
        U["用户 / 浏览器"]
        WEB["web/\nReact + TypeScript\n页面、路由、组件、hooks"]
    end

    subgraph L2["API 网关层"]
        API["Go / Python API 入口\nGo: internal/router\nPython: api/apps"]
        AUTH["鉴权 / 参数校验 / 路由分发"]
    end

    subgraph L3["服务层"]
        SVC["业务服务\ninternal/service\napi/db/services"]
        DAO["数据访问\ninternal/dao\napi/db/db_models"]
        TASK_CREATE["创建解析任务\n写任务状态 / 推送队列"]
    end

    subgraph L4["核心引擎层"]
        TE["rag/svr/task_executor.py\n异步任务消费"]
        CHUNKER["rag/app/*\n按 parser_id 选择分块策略"]
        PARSER["解析后端\nDeepDOC 默认/内置主力\nMinerU / Docling / PaddleOCR 等可选"]
        EMB["Embedding 模型"]
        SEARCH["rag/nlp/search.py Dealer\n全文 + 向量混合检索"]
        RERANK["Rerank 模型"]
        LLM["rag/llm\nLLM 生成 / 模型封装"]
        AGENT["agent/\nAgent 编排、工具、沙箱"]
    end

    subgraph L5["存储层"]
        MYSQL["MySQL\n用户、租户、知识库、文档、任务、会话元数据"]
        OBJ["MinIO / S3 / OSS\n原始文件、图片、附件"]
        REDIS["Redis\n任务队列、缓存、分布式锁"]
        IDX["ES / Infinity / OceanBase\n全文索引 + 向量索引"]
    end

    U --> WEB
    WEB --> API
    API --> AUTH
    AUTH --> SVC
    SVC --> DAO
    DAO --> MYSQL

    SVC --> OBJ
    SVC --> TASK_CREATE
    TASK_CREATE --> REDIS

    REDIS --> TE
    TE --> CHUNKER
    CHUNKER --> PARSER
    PARSER --> CHUNKER
    CHUNKER --> EMB
    EMB --> IDX
    TE --> MYSQL

    WEB --> SEARCH
    API --> SEARCH
    SEARCH --> IDX
    SEARCH --> RERANK
    RERANK --> LLM
    LLM --> WEB

    API --> AGENT
    AGENT --> LLM
    AGENT --> SEARCH
    AGENT --> MYSQL
    AGENT --> REDIS
``````











本文把这个拆解为两个链路进行分析。


