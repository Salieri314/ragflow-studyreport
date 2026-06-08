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

## 一\. 上传文件与元数据初始化

源码入口主要在：

- api/apps/restful\_apis/document\_api\.py \(line 370\)

- api/db/services/file\_service\.py \(line 458\)

- api/db/db\_models\.py \(line 904\)

整体的流程是

```ABAP
前端上传文件
→ /datasets/<dataset_id>/documents
→ 校验知识库权限
→ 读取 multipart 文件
→ FileService.upload_document()
→ 原始文件写入对象存储
→ MySQL 写 document/file/file2document
→ 返回 document 列表给前端
```

### API入口先判断上传类型

document\_api\.py 里上传接口支持几种类型：local 本地上传，web url转PDF， empty 空文档

本地上传最后进入\_upload\_local\_documents\(kb, tenant\_id\)

关键逻辑在document\_api\.py \(line 572\)

```ABAP
读取 request.form
读取 request.files
检查 file 是否存在
检查文件名长度
解析可选 parser_config
调用 FileService.upload_document()
```

这里有一个细节：上传时允许传 parser\_config 覆盖项，但代码只允许少数表格相关字段：

```ABAP
allowed_keys = {"table_column_mode", "table_column_roles"}
```

这说明上传接口本身不允许用户随便覆盖解析配置，防止前端把任意 parser 配置塞进来。



### FileService\.upload\_document 负责真正落库

先处理文件目录结构：

```ABAP
获取用户根目录
初始化 knowledgebase_docs 目录
获取知识库文件夹
为当前 KB 创建/获取文件夹
处理 parent_path
```

合并 parser 配置

```ABAP
base_parser_config = kb.parser_config or {}
merged_parser_config = {**base_parser_config, **parser_config_override}
```

也就是说，文档默认继承知识库的：

```ABAP
kb.parser_id
kb.pipeline_id
kb.parser_config
```

但上传时允许有限覆盖。

### 原始文件进入对象存储

对每个文件，代码会生成 doc\_id，检查重名和文件类型。

关键字段：

```ABAP
filename = duplicate_name(...)
filetype = filename_type(filename)
location = filename or parent_path/filename
```

然后读取文件二进制：

```ABAP
blob = file.read()
```

如果是pdf，会先尝试修复潜在的损坏：

```ABAP
if filetype == FileType.PDF.value:
    blob = read_potential_broken_pdf(blob)
```

之后写对象存储

```ABAP
settings.STORAGE_IMPL.put(kb.id, location, blob)
```

这里的 kb\.id 可以理解为对象存储 bucket 或命名空间，location 是该文件在对象存储里的路径。

所以此时：

```ABAP
MySQL 还没有 chunk
向量库还没有数据
对象存储已经有原始文件
```

### 生成缩略图

上传时还会尝试生成缩略图：

```ABAP
img = thumbnail_img(filename, blob)
thumbnail_location = f"thumbnail_{doc_id}.png"
settings.STORAGE_IMPL.put(kb.id, thumbnail_location, img)
```

缩略图也是放对象存储，document\.thumbnail 保存的是位置

### 创建 Document 元数据

随后构造 doc 字典：

```Python
doc = {
    "id": doc_id,
    "kb_id": kb.id,
    "parser_id": self.get_parser(filetype, filename, kb.parser_id),
    "pipeline_id": kb.pipeline_id,
    "parser_config": merged_parser_config,
    "created_by": user_id,
    "type": filetype,
    "name": filename,
    "source_type": src,
    "suffix": Path(filename).suffix.lstrip("."),
    "location": location,
    "size": len(blob),
    "thumbnail": thumbnail_location,
    "content_hash": xxhash.xxh128(blob).hexdigest(),
}
DocumentService.insert(doc)
```

这对应 MySQL 的 document 表，定义在 db\_models\.py \(line 904\)。

最关键字段是：

```ABAP
id              文档 ID
kb_id           所属知识库
parser_id       分块策略类型
pipeline_id     是否走自定义 dataflow
parser_config   解析/分块配置
type            文件类型
name            文件名
location        对象存储位置
size            文件大小
content_hash    内容哈希，用于变更检测
progress/run    解析进度与状态
```

这里特别重要的是 parser\_id 的计算：

```ABAP
self.get_parser(filetype, filename, kb.parser_id)
```

也就是说文档不一定完全使用知识库默认 parser。特殊文件会被强制改：

```ABAP
图片      → picture
音频      → audio
PPT/pages → presentation
eml/msg   → email
其他      → kb.parser_id
```

这一步决定了后面 task\_executor 会调用哪个 rag/app/\*\.py

### 创建 File 与 File2Document

RAGFlow 里 document 和 file 是两个概念

```ABAP
document:
知识库中的可解析文档对象

file:
文件管理器里的文件/文件夹对象
```

上传后会调用：

```ABAP
FileService.add_file_from_kb(doc, kb_folder["id"], kb.tenant_id)
```

它会写：

```ABAP
file
file2document
```

file2document 是文件管理器文件和知识库 document 的关联表。

所以这一步之后，MySQL 至少新增/更新：

```ABAP
document
file
file2document


document 用于：
- 解析状态
- parser_id
- parser_config
- chunk 数量
- token 数量
- RAG 检索
- 删除索引

file 用于：
- 文件夹结构
- 文件列表展示
- 移动文件
- 文件管理器
- 关联外部数据源文件

file2document 用于：
- 建立 file 和 document 的映射
```

举个直观的例子

用户在前端看到：

```ABAP
文件管理器
公司制度库/
  ├─ 员工手册.pdf
  └─ 报销制度.docx
```

这个目录结构来自 file 表

但当用户点击“解析员工手册\.pdf”时，系统要找的是：员工手册\.pdf 对应哪个 document？

于是查 file2document

file\_001 → doc\_001







对象存储新增：

```ABAP
原始文件
可能的 thumbnail
```

这一阶段不会立即生成向量，这是很多人容易误解的地方。上传完成后，返回前端的状态是：

```ABAP
run_status = "0"
```

也就是还没解析。上传阶段只是完成：

```ABAP
原始文件保存
元数据登记
文件系统关系建立
parser_id/parser_config 确定
```

真正创建 task、推送 Redis、开始解析，是后面的 /documents/parse 或 DocumentService\.run\(\) 触发的。

在文档入库链路的第一阶段，RAGFlow 首先通过 API 接收用户上传的文件，并将文件二进制内容写入对象存储，同时在 MySQL 中创建 document、file 和 file2document 等元数据记录。此阶段会根据知识库默认配置和文件类型确定 document\.parser\_id 与 parser\_config，但不会立即执行 OCR、分块或 embedding。上传完成后，文档处于待解析状态，后续由解析接口将其转化为异步任务并推入 Redis 队列。



## 二\. 创建解析任务并且推入Redis 队列

第一阶段结束时，系统里已经有：

```ABAP
对象存储：原始文件
MySQL：document / file / file2document
```

但还没有真正解析，也没有向量索引。第二阶段的目标是：把 document 转成后台可执行的解析 task，并推入 Redis 队列。



### 前端触发解析

上传完成后，前端通常会再调用解析接口：

```ABAP
POST /datasets/<dataset_id>/documents/parse
```

源码入口在：

api/apps/restful\_apis/document\_api\.py \(line 1430\)

核心函数：async def parse\_documents\(tenant\_id, dataset\_id\):

它接收：

```ABAP
{
  "document_ids": ["doc_001", "doc_002"]
}
```

也就是说，前端告诉后端请开始解析这些 document。



### API 校验 document 是否属于当前知识库

代码会检查：

```ABAP
docs = DocumentService.query(kb_id=dataset_id, id=doc_id)
```

意思是：

```ABAP
这个 doc_id 必须属于当前 dataset_id。
```

这样可以避免用户拿别的知识库的 document\_id 来触发解析。

如果找不到，会返回：

```ABAP
Documents not found
```

### 更新 document 状态为 RUNNING

对每个有效 document，代码会设置：

```ABAP
info = {
    "run": str(TaskStatus.RUNNING.value),
    "progress": 0
}
DocumentService.update_by_id(doc_id, info)
```

这里的 run 是文档解析状态。

常见状态来自 TaskStatus：

```ABAP
0  UNSTART
1  RUNNING
2  CANCEL
3  DONE
4  FAIL
5  SCHEDULE
```

所以这一步表示

```ABAP
这个 document 已经准备进入解析流程。
```

如果是重新解析已经完成的文档，还会清空旧统计：

```ABAP
info["progress_msg"] = ""
info["chunk_num"] = 0
info["token_num"] = 0
```

### 删除旧 task 和旧索引

重新解析时，代码会先清理旧任务：

```ABAP
TaskService.filter_delete([Task.doc_id == doc_id])
```

然后如果索引存在，会删除旧 chunk：

```ABAP
settings.docStoreConn.delete(
    {"doc_id": doc_id},
    search.index_name(tenant_id),
    doc.kb_id
)
```

这一步的意义是：

```ABAP
避免旧 chunk 和新 chunk 混在一起。
```

比如你重新上传了新版《员工手册\.pdf》，旧版本的向量索引就应该删除，否则问答时可能召回过期内容。

备注：重新上传知识doc\_id必须一样，如果只是单纯上传同名文件，程序会避免重名而将其改名为员工手册（1），而进行全新的解析。



### 调用 DocumentService\.run\(\)

api/db/services/document\_service\.py \(line 1045\)

```ABAP
DocumentService.run(tenant_id, doc_dict, kb_table_num_map)
```

这个函数决定：

这个 document 是走普通解析任务，还是走 dataflow pipeline。

源码逻辑是：

```ABAP
if doc.get("pipeline_id", ""):
    queue_dataflow(...)
else:
    bucket, name = File2DocumentService.get_storage_address(doc_id=doc["id"])
    queue_tasks(doc, bucket, name, 0)
```

也就是说有两条分支：

```ABAP
如果 document 有 pipeline_id
→ 走自定义 dataflow

如果没有 pipeline_id
→ 走普通 queue_tasks 文档解析
```

我们先分析普通路径，也就是

```ABAP
queue_tasks(doc, bucket, name, 0)
```

### 通过 file2document 找对象存储位置

这里为什么要查？

因为 task\_executor 后面需要知道

```ABAP
原始文件在哪个 bucket？
对象存储里的 location 是什么？
```

第一阶段我们说过：

```ABAP
file 表保存 location
document 表也有 location
file2document 把 file 和 document 绑定
```

这里就是通过 document 找到对应文件的存储地址。

### queue\_tasks 创建 Task

核心函数在：

api/db/services/task\_service\.py \(line 356\)

```ABAP
def queue_tasks(doc: dict, bucket: str, name: str, priority: int):
```

它负责：

```ABAP
根据 document 创建一个或多个 task
写入 task 表
推入 Redis 队列
```

### 为什么一个 document 可能变成多个 task？

这点很重要。

如果是普通小文件：

```ABAP
一个 document → 一个 task
```

但如果是 PDF，RAGFlow 会按页范围切 task：

```SQL
if doc["type"] == FileType.PDF.value:
    pages = PdfParser.total_page_number(...)
    page_size = doc["parser_config"].get("task_page_size") or 12
    ...
    for p in range(s, e, page_size):
        task["from_page"] = p
        task["to_page"] = min(p + page_size, e)
```

默认情况下，一个 PDF 每 12 页一个 task。

比如一个 35 页 PDF：

```ABAP
task 1: page 0 ~ 12
task 2: page 12 ~ 24
task 3: page 24 ~ 35
```

为什么这么做？

```ABAP
1. 大 PDF 可以分段处理
2. OCR/解析失败时只影响局部页
3. 多 worker 可以并行
4. 进度更容易更新
```

但有例外：

```SQL
if doc["parser_id"] in ["one", "knowledge_graph"] 
or do_layout != "DeepDOC" 
or doc["parser_config"].get("toc_extraction", False):
    page_size = MAXIMUM_TASK_PAGE_NUMBER
```

意思是：

```ABAP
如果是整篇解析、知识图谱、非 DeepDOC 后端、目录提取等情况，
就不按 12 页拆，而是整个文档一个大 task。
```

### 表格文档按行切 task

如果文档 parser 是 table：

```SQL
elif doc["parser_id"] == "table":
    rn = RAGFlowExcelParser.row_number(...)
    for i in range(0, rn, 3000):
        task["from_page"] = i
        task["to_page"] = min(i + 3000, rn)
```

也就是说表格不是按页切，而是按行切：

每 3000 行一个 task

一定注意，这个是切task，不是文本切分，这是两个概念！！！！

### 每个 task 的基础字段

每个 task 大概长这样：

```JSON
{
    "id": get_uuid(),
    "doc_id": doc["id"],
    "progress": 0.0,
    "from_page": 0,
    "to_page": MAXIMUM_TASK_PAGE_NUMBER,
    "begin_at": 当前时间
}
```

然后还会补：

```ABAP
digest
priority
```

digest 是一个哈希，用来判断任务配置是否变化。



### task digest 的作用

代码会基于：

```ABAP
chunking_config
doc_id
from_page
to_page
```

计算：

```ABAP
task["digest"] = xxhash.xxh64(...)
```

这个 digest 可以帮助复用旧任务结果。

如果用户重新解析，但配置没变、页范围没变，系统可能复用之前已经完成的 chunk，减少重复计算。

代码里有：

```ABAP
reuse_prev_task_chunks(task, prev_tasks, chunking_config)
```

这是一个优化点



### 写入 task 表

所有 task 准备好后：

```ABAP
bulk_insert_into_db(Task, parse_task_array, True)
```

这一步写 MySQL 的 task 表。

task 表记录：

```ABAP
task.id
task.doc_id
task.from_page
task.to_page
task.progress
task.progress_msg
task.digest
task.chunk_ids
task.priority
```

这时任务已经持久化，即使服务重启也能知道任务存在过。

### 更新 document 为 queued

随后调用：

```ABAP
DocumentService.begin2parse(doc["id"])
```

源码在 document\_service\.py \(line 891\)

它会更新：

```ABAP
{
    "progress_msg": "Task is queued...",
    "process_begin_at": 当前时间,
    "progress": 一个很小的随机值,
    "run": RUNNING
}
```

所以前端看到的状态会变成类似：

```ABAP
Task is queued...
```

### 推入 Redis 队列

最后，把未完成的 task 推到 Redis：

```ABAP
REDIS_CONN.queue_product(
    settings.get_svr_queue_name(priority, suffix),
    message=unfinished_task
)
```

这里的 suffix 根据 parser\_id 决定：

```ABAP
suffix = "common" if doc["parser_id"] != "resume" else "resume"
```

意思是：

```ABAP
普通任务 → common 队列
简历任务 → resume 队列
```

这样可以把特别复杂或耗时的 resume 解析和普通任务隔离。

### 第二阶段结束后的系统状态

到这里，系统状态变成：

```ABAP
MySQL:
document.run = RUNNING
document.progress ≈ 0
document.progress_msg = Task is queued...
task 表新增一条或多条 task

Redis:
对应 task message 已推入队列

对象存储:
仍然保存原始文件

索引库:
可能已经删除旧 chunk
新 chunk 还没有生成
```

本阶段总结

```ABAP
第二阶段完成的是“调度准备”，不是实际解析。

它把一个 document 拆成一个或多个 task，
写入 MySQL task 表，
并推送到 Redis 队列，
等待 rag/svr/task_executor.py 消费。
```

一定注意，这个是切task，不是文本切分，这是两个概念！！！！

Redis 里放的是什么？edis 队列里不是文件内容，也不是 chunk，而是类似这样的任务消息：

```JSON
{
  "id": "task_001",
  "doc_id": "doc_001",
  "from_page": 0,
  "to_page": 12,
  "progress": 0.0,
  "priority": 0,
  "digest": "xxxx"
}
```

这个 task 只告诉后台：请处理 doc\_001 的第 0 到 12 页。

后台 task\_executor 收到后，才会去：

```ABAP
查 MySQL 拿 document 配置
去对象存储读原始文件
调用 parser/chunker
生成 chunk
embedding
写索引
```

为什么要先拆 task？

主要是为了：

```ABAP
大文件并行处理
减少单个任务失败影响
方便进度追踪
支持重试
支持队列调度
支持 resume 等特殊队列隔离
```

在解析触发阶段，RAGFlow 并不会直接在 API 请求中执行文档解析，而是先将 document 标记为 RUNNING，并根据文件类型、parser\_id 和 parser\_config 创建一个或多个 Task。PDF 会按页范围拆分，表格会按行范围拆分，普通文档通常生成单个任务。Task 被写入 MySQL 后，再通过 Redis 队列投递给后台 task\_executor。该设计将用户请求与耗时解析解耦，并为并行处理、失败重试、进度追踪和旧任务复用提供基础。



### 补充说明：Redis工作原理

相信还是有一些小伙伴和我一样学校的时候一笔带过，在真正做项目之前并不了解Redis的，这里简单讲解一些Redis是什么。

Redis 可以先理解成一个 **运行在内存里的高速数据仓库**。

但在 RAGFlow 里，最重要的是它被用来做：

```ABAP
任务队列
缓存
分布式锁
运行状态协调
```

你可以把 Redis 想成一个“后台任务收发室”。

为什么需要 Redis？

如果用户上传一个 200 页 PDF，解析、OCR、embedding 可能要很久。

如果 API 直接在用户请求里做这些事，就会变成：

```ABAP
用户点击上传
→ 浏览器一直等
→ 后端一直卡着
→ 超时或体验很差
```

所以 RAGFlow 采用异步方式：

```ABAP
API 不直接解析文件
API 只创建任务
然后把任务丢给 Redis
后台 worker 慢慢消费
```

Redis 在这里像什么？可以想象成快递驿站：

```ABAP
API 服务
= 发件人

Redis
= 快递驿站 / 任务排队区

task_executor
= 快递员 / 后台工人

任务 task
= 包裹
```

流程是：

```ABAP
API 创建 task
→ 把 task 放进 Redis 队列

task_executor 从 Redis 队列取 task
→ 处理文档解析
→ 处理完成后更新数据库
```

Redis 里放的不是文件本身，这点很重要。

Redis 里通常不放 PDF 原始内容，也不放完整 chunk。它放的是任务消息，比如：

```JSON
{
  "id": "task_001",
  "doc_id": "doc_001",
  "from_page": 0,
  "to_page": 12,
  "priority": 0
}
```

意思是：请后台处理 doc\_001 的第 0 到 12 页。

真正的文件在：MinIO / S3 / OSS 对象存储

真正的文档元数据在：MySQL

真正的向量索引在：ES / Infinity / OceanBase

Redis 只是让这些组件协同起来。



RAGFlow 里 Redis 的作用

```ABAP
1. 任务队列
   文档解析任务进入 Redis，task_executor 消费。

2. 缓存
   一些模型结果、查询结果、配置可能缓存起来，减少重复计算。

3. 分布式锁
   多个后台 worker 同时运行时，防止重复执行某些全局操作。

4. 状态协调
   队列长度、未完成任务、消费者状态等。
```

一句话总结

Redis 在 RAGFlow 里不是“存知识库内容”的地方，而是 让后台任务排队和协调执行的高速中间层。

放到链路一里就是：

```ABAP
MySQL 记录：有这么一个任务
Redis 负责：把这个任务交给后台 task_executor 去执行
```

