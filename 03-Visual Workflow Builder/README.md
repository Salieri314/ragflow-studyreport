# 链路三：可视化编排

这条线回答的是：

RAGFlow 如何像 Dify 一样，让用户通过拖拽节点、连线、配置参数，构造一个可执行的 Agent 或 Dataflow 流程？

---

第三条线总览

```YAML
用户在前端画布拖拽节点
  |
  v
前端维护 nodes / edges / form
  |
  v
前端生成 DSL JSON
  |
  v
后端保存 DSL 到 MySQL
  |
  v
运行时读取 DSL
  |
  v
Graph / Canvas / Pipeline 解释 DSL
  |
  v
根据 component_name 动态加载组件
  |
  v
按 upstream / downstream 执行节点
  |
  v
节点 output 传给下游 input
  |
  v
记录 Redis trace / 保存会话 / 返回结果
```

可以画成这样：

```ABAP
┌──────────────────────────────┐
│ 1. 前端画布层                 │
│ web/src/pages/agent           │
│ nodes / edges / form          │
└───────────────┬──────────────┘
                │
                │ buildDslData()
                v
┌──────────────────────────────┐
│ 2. DSL 构造层                 │
│ graph: 给前端还原画布          │
│ components: 给后端执行流程      │
└───────────────┬──────────────┘
                │
                │ POST/PUT Agent
                v
┌──────────────────────────────┐
│ 3. DSL 存储层                 │
│ MySQL user_canvas.dsl         │
│ MySQL user_canvas_version.dsl │
│ Redis canvas replica          │
└───────────────┬──────────────┘
                │
                │ run / debug / chat
                v
┌──────────────────────────────┐
│ 4. DSL 解释层                 │
│ agent/canvas.py Graph/Canvas  │
│ rag/flow/pipeline.py Pipeline │
│ normalize_chunker_dsl         │
└───────────────┬──────────────┘
                │
                │ component_name -> class
                v
┌──────────────────────────────┐
│ 5. 动态组件加载层             │
│ component_class()             │
│ Agent / Retrieval / Generate  │
│ Parser / Chunker / Extractor  │
│ Tool / Switch / Loop          │
└───────────────┬──────────────┘
                │
                │ invoke / invoke_async
                v
┌──────────────────────────────┐
│ 6. 节点执行与数据流层          │
│ upstream / downstream         │
│ output() -> invoke(**kwargs)  │
│ sys.query / env.xxx / cpn@var │
└───────────────┬──────────────┘
                │
                │ callback / trace
                v
┌──────────────────────────────┐
│ 7. 日志、错误与结果层          │
│ Redis trace logs              │
│ _ERROR / exception_goto       │
│ conversation.dsl / reference  │
└──────────────────────────────┘
```

---

这条线里有两个核心运行场景。

场景一：Agent Canvas

用于构建对话型 Agent。

比如：

```ABAP
Begin
  |
  v
Retrieval
  |
  v
Generate
  |
  v
Message
```

或者更复杂：

```ABAP
Begin
  |
  v
Switch / Categorize
  |             |
  v             v
Retrieval      Tool
  |             |
  v             v
Generate ----> Message
```

它主要运行在：

```ABAP
agent/canvas.py
agent/component/
agent/tools/
```

---

场景二：Dataflow / Pipeline

用于构建文档处理流水线。

比如：

```ABAP
File
  |
  v
Parser
  |
  v
TokenChunker
  |
  v
Extractor
  |
  v
Output
```

它主要运行在：

```ABAP
rag/flow/pipeline.py
rag/flow/
```

这条线和第一条知识库线有交叉：

如果某个知识库绑定了自定义 Dataflow，文档解析可以不走普通 parser\_id \-\> chunker，而走：

```ABAP
DSL Pipeline -> File / Parser / Chunker / Extractor 节点
```

---

```ABAP
1. 前端画布模型
   nodes / edges / form
   用户拖拽和连线如何表示成图

2. DSL 生成
   buildDslData()
   graph 与 components 的区别
   upstream / downstream 如何生成

3. DSL 保存与版本
   agent_api.py
   user_canvas.dsl
   user_canvas_version.dsl
   canvas_template.dsl
   Redis replica

4. 运行入口
   Agent chat / debug / webhook
   Dataflow run
   后端如何选择 Canvas 或 Pipeline

5. DSL 解释与初始化
   Graph.__init__()
   normalize_chunker_dsl
   Graph.load()
   component 参数校验

6. 动态加载组件
   component_class()
   component_name -> Param / Component
   插件式组件扩展

7. 节点执行与数据流
   path
   upstream / downstream
   output() -> invoke()
   变量引用 sys.xxx / env.xxx / component@field
   Switch / Categorize / Loop / UserFillUp

8. 日志、错误与结果保存
   Redis trace
   _ERROR
   exception_default_value / exception_goto
   tool_use_callback
   conversation.dsl / final answer
```

---

一句话概括第三条线：

**第三条线就是 RAGFlow 的可视化编排引擎：前端把画布转换成 DSL，后端把 DSL 解释成组件对象，再按连线执行组件，组件之间通过 output/input 传递数据，运行过程通过 Redis trace 和 conversation 保存下来。**

