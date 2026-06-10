# 链路二：用户提问到 RAG 回答链路。

这条线对应的是：

```YAML
用户问题
  |
  v
后端 Chat/Search/Agent API
  |
  v
读取会话、知识库、模型配置
  |
  v
检索 chunks
  |
  v
rerank / 引用整理
  |
  v
构造 prompt
  |
  v
LLM 生成回答
  |
  v
返回前端并保存会话
```

我将第二条线分成 7 个大模块来讲：

```ABAP
1. Chat 请求入口
   前端发起问题，后端进入 chat_api.py

2. 会话与配置加载
   读取 dialog、conversation、knowledgebase、模型配置、prompt 配置

3. 问题预处理与检索参数构造
   历史消息、query、kb_ids、top_k、similarity_threshold 等

4. 混合检索
   rag/nlp/search.py Dealer.retrieval()
   全文检索 + 向量检索 + metadata filter

5. rerank 与引用整理
   rerank 模型重排 chunks，整理 reference / citation / doc_aggs

6. prompt 构造与 LLM 生成
   把 chunks 拼进 prompt，调用 LLMBundle，支持流式返回

7. 返回前端与保存会话
   answer、reference、message、tokens、duration 写回 MySQL
```

第一条线是“把文档变成知识”；第二条线是“把问题变成答案”。
