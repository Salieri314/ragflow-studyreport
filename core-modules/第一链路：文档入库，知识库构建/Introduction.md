# 链路1：文档入库 / 知识库构建链路

一个用户上传的文件，如何变成可以被RAG检索的知识？

完整链路：

```ABAP
用户上传文件
→ web 前端
→ Go / Python API
→ MySQL 写 file / document / task 元数据
→ MinIO / S3 / OSS 保存原始文件
→ Redis 推送解析任务
→ rag/svr/task_executor.py 消费任务
→ rag/app 根据 parser_id 选择分块策略
→ DeepDOC / MinerU / Docling / PaddleOCR 等解析后端提取内容
→ 生成 chunks
→ Embedding 模型生成向量
→ ES / Infinity / OceanBase 写入全文索引和向量索引
→ MySQL 更新文档进度、chunk 数、token 数
```

这条链路的核心目标是

```ABAP
原始文件
→ 结构化内容
→ chunk
→ embedding
→ 可检索索引
```
