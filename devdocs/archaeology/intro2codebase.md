# 代码库简介：架构导览

欢迎。本项目是一个 JavaScript/TypeScript SDK，封装了 Bright Data 的网页爬虫、搜索和代理 API。阅读代码无需了解产品本身，重要的是代码如何组织，而本文档正是对此进行介绍。

## 30 秒概览

SDK 由五个同心层组成。数据和控制流从用户向外流向网络：

```
┌────────────────────────────────────────────────────────────┐
│ Facade:           bdclient (src/client.ts)                 │  ← the one public class
├────────────────────────────────────────────────────────────┤
│ Feature surfaces: scrape / search / datasets / discover /  │  ← domain-grouped APIs
│                   scraperStudio / browser / zones          │
├────────────────────────────────────────────────────────────┤
│ Async orchestration: *Job classes + polling + Deadline     │  ← long-running operations
├────────────────────────────────────────────────────────────┤
│ Validation / models: zod schemas + Result classes          │  ← boundary + value objects
├────────────────────────────────────────────────────────────┤
│ Transport:        Transport + RateLimiter + undici         │  ← a single shared HTTP client
└────────────────────────────────────────────────────────────┘
```

用户执行的所有操作都会经过 `bdclient` 外观，它会延迟构建各个功能接口。所有接口共享一个用于 HTTP 的 `Transport` 实例、一个速率限制器和一组身份验证标头。

## 主要抽象（你会在各处看到的名词）

**`bdclient`**（`src/client.ts`）。用户唯一需要实例化的类。它读取选项和环境变量，构建 `Transport`，并通过一个小型 `defineLazy` 辅助函数，在首次访问时延迟实例化每个子 API。其他所有内容都挂载在此对象的属性上：`client.scrape.linkedin`、`client.search.google`、`client.datasets.amazonProducts`、`client.discover`、`client.scraperStudio`、`client.browser`、`client.listZones()`。

**`Transport`**（`src/core/transport.ts`）。唯一的核心 HTTP 客户端。它封装 undici，并提供连接池、keep-alive、重试拦截器、DNS 缓存、可选的速率限制器，以及统一的 `classifyError` 步骤，将 undici 错误转换成 SDK 自己的类型化错误层级。代码库中所有 HTTP 通信都经过这里，不存在零散的 `fetch` 调用。

**`RateLimiter`**（`src/core/rate-limiter.ts`）。带等待队列的令牌桶。`Transport.request` 会在每个出站请求前调用 `acquire()`。它是可选功能（默认关闭），可通过客户端上的 `rateLimit` 启用。

**功能“路由器”和“服务”**。每个功能系列都有一个由客户端公开的小型分发器类：
- `ScrapeRouter`（`src/api/scrape/router.ts`）：为每个平台爬虫各持有一个实例（`linkedin`、`amazon`、`instagram`、`facebook`、`tiktok`、`youtube`、`reddit`、`pinterest`、`digikey`、`chatGPT`、`perplexity`），并持有一个由它们共享的 `SnapshotAPI`。
- `SearchRouter`：所有 `google / bing / yandex` 方法都委托给同一个 `SearchAPI`。
- `DatasetsClient`：125 个以上的延迟 getter，每个对应一个预采集数据集。
- `DiscoverService`、`ScraperStudioService`、`BrowserService`：独立的功能服务。
- `ZonesAPI`：不是路由器，只是一个缓存的 API 包装器。

**`*API` 类**（`src/api/**/*.ts`）。真正负责构造请求的主力。它们分为两个系列：
- `RequestAPI`（位于 `src/api/unlocker/` 中）：由 `ScrapeAPI` 和 `SearchAPI` 继承。同步风格的端点（`POST /request`）。数组并发通过 `@supercharge/promise-pool` 实现。基类定义流程，子类覆盖 `getURL` 和 `getMethod`（模板方法模式）。
- `BaseAPI`（位于 `src/api/scrape/` 中）：由平台爬虫（`LinkedinAPI`、`AmazonAPI`、……）以及 `SnapshotAPI` 继承。异步、面向数据集的端点（`POST /datasets/v3/trigger|scrape`）。平台子类主要使用类型化输入和硬编码的 `datasetId` 封装 `run()` 或 `orchestrate()`。

**`*Job` 类**（`ScrapeJob`、`DiscoverJob`、`ScraperStudioJob`）。用于承载长期运行的服务端任务的值句柄。它们具有相同的结构：一个 ID、触发时间戳、一组 `status()` / `wait()` / `fetch()` / `toResult()` 方法（或等效方法），以及状态和结果的内部缓存。在不进行编排的情况下触发异步操作时，你会获得这类对象。

**`*Result` 类**（`BaseResult`、`ScrapeResult`、`SearchResult`、`DiscoverResult`）。从 `job.toResult()` 返回的值对象。它们永远不会抛出异常：失败会进入一个 `success: false` 结果，其中包含 `error`、`status` 和计时字段。它们可通过 `toJSON()` 干净地序列化，也可以通过 `save()` 将自身写入磁盘。

**`BaseDataset`**（`src/api/datasets/base.ts`）。所有 125 个数据集类的抽象父类。每个具体子类实际上只有 3 行（一个硬编码的 `datasetId` 和 `name`）。基类针对 `/datasets/*` 端点提供 `query()`、`sample()`、`getStatus()`、`download()`，这是典型的仓库模式。

**Zod schema**（`src/schemas/`）。SDK 在边界处执行验证。每个公共方法都以 `assertSchema(...)` 调用开始，将用户输入转换成类型化、规范化的数据。响应解析使用 `parseResponse` 对照响应结构 schema 进行验证，因此服务端契约也会得到检查。`ValidationError` 是这些调用产生的唯一异常类型。

**错误层级**（`src/utils/errors.ts`）。所有错误都继承自 `BRDError`。具体类型包括：`ValidationError`、`AuthenticationError`、`ZoneError`、`NetworkError` → `NetworkTimeoutError`、`TimeoutError`、`FSError`、`APIError`（携带 `statusCode`）、`DataNotReadyError`。`Transport.classifyError` 和 `throwInvalidStatus` 是决定抛出何种类型的两个关键节点。

**`Deadline`**（`src/utils/deadline.ts`）。一个小型辅助工具，使用 `.remaining` 和 `.expired` 封装 `start + budget`。`*Job.toResult()` 风格的方法使用它来确保多次重试共享一个总时间预算，这样竞态条件重试就不会每次都重置新的超时时间。

## 主要流程如何在系统中运转

### 1) 简单同步抓取（网络解锁器）

```
client.scrapeUrl(url)
  → assertSchema(URLParamSchema / ScrapeOptionsSchema)
  → ScrapeAPI.handle (extends RequestAPI)
      → optionally ZonesAPI.ensureZone (if autoCreateZones)
      → handleSingle or handleBatch (PromisePool, concurrency=10 default)
          → Transport.request(POST /request, body={url, zone, format, method, …})
              → RateLimiter.acquire()
              → undici.request(…)
              → classifyError on failure
          → assertResponse → parse body as text or JSON
  → string | object | array of either
```

搜索（`client.search.google(q)`）使用相同的管道，只是采用不同的 `*API` 子类（`SearchAPI`），将查询放入特殊的 URL 格式中。

### 2) 异步平台抓取（数据集 v3）

```
client.scrape.linkedin.profiles(urls)
  → assertSchema
  → LinkedinAPI.orchestrate(input, DATASET_ID.PROFILE)
      → BaseAPI.run(async: true) → POST /datasets/v3/trigger → { snapshot_id }
      → new ScrapeJob(snapshot_id, snapshotOps)
      → job.toResult({ pollTimeout, pollInterval })
          → Deadline(pollTimeout)
          → wait() → pollUntilReady(snapshot_id, SnapshotAPI.getStatus)
              → GET /datasets/v3/progress/{id} every pollInterval
              → resolves when status === 'ready'
          → fetch() → GET /datasets/v3/snapshot/{id}
              → 202 → throw DataNotReadyError → retry inside toResult()
              → parse as JSON
  → ScrapeResult(success=true, data, rowCount, timing…)
```

原始 `collect*` / `discover*` 方法（例如 `client.scrape.linkedin.collectProfiles`）会更早返回：它们会提供一个由你自行编排的 `ScrapeJob`；如果服务器在同步超时前完成，则返回同步模式数据。`orchestrate()` 只是“一次调用完成全部操作”的便捷方法。

### 3) Discover（AI 驱动的搜索）

```
client.discover(query)
  → DiscoverService.search → _trigger → POST /discover → { task_id }
  → new DiscoverJob(task_id, ops)
  → job.toResult()
      → wait() loops GET /discover?task_id=… every 2s until status === 'done'
      → results come back on the status response itself (no separate fetch endpoint)
  → DiscoverResult(success, data=items, totalResults, durationSeconds)
```

注意它与 ScrapeJob 的细微区别：Discover 的“完成”状态会附带有效负载，无需单独调用下载。

### 4) 爬虫工作室（DCA）

```
client.scraperStudio.run(collector, { input })
  → POST /dca/trigger_immediate?collector=… → { response_id }
  → new ScraperStudioJob(response_id, transport)
  → job.waitAndFetch()
      → loops GET /dca/get_result?response_id=…
      → 202 → DataNotReadyError → sleep → retry (up to Deadline)
      → 200 → parse JSON array, return
  → collected into RunResult[] with per-input success/error tracking
```

### 5) 预采集数据集

```
client.datasets.amazonProducts.query({ asin: 'B0…' })
  → BaseDataset.query → POST /datasets/filter → { snapshot_id }
  → client.datasets.amazonProducts.download(snapshot_id)
      → pollUntilReady loops GET /datasets/snapshots/{id} until ready
      → GET /datasets/snapshots/{id}/download → parse JSON array
```

这里有一个值得尽早了解的命名空间冲突：存在**两套不同的“数据集”系统**。`client.scrape.*` 调用 Bright Data 的 `/datasets/v3/` 爬虫端点（你提供输入，由它运行采集器）。`client.datasets.*` 调用另一套 `/datasets/` 预采集端点（你查询现有数据集）。除传输层外，两者不共享代码。

### 6) 浏览器 API

这是一个特例：完全没有 HTTP 调用。`client.browser.getConnectUrl({ country })` 返回一个 `wss://…` 字符串，你可以将其传入 Puppeteer/Playwright，后者会自行建立实际连接。凭据来自构造函数选项或 `BRIGHTDATA_BROWSERAPI_*` 环境变量，并在首次访问时延迟验证。

## 使用的设计模式

- **外观模式**：`bdclient` 将五个不同的功能系列隐藏在一个易用对象之后。
- **分层架构**：用户代码 → 功能服务/路由器 → API 类 → Transport → undici。任何层都不会跨层向下调用。
- **模板方法**：`RequestAPI`（覆盖项：`getURL`、`getMethod`）和 `BaseAPI.run/orchestrate`（子类仅使用各自的 `datasetId` 调用它）。
- **仓库模式**：`BaseDataset` 及约 125 个单行子类，每个子类都是特定远程数据集的仓库。
- **命令 + Future（Job）**：`ScrapeJob`、`DiscoverJob`、`ScraperStudioJob`。触发操作会返回一个句柄；该句柄公开轮询/获取/结果操作。每个对象也都有一个便捷的 `toResult()` 方法，将成功和失败封装为 Result 值对象。
- **值对象**：`BaseResult` 层级。不可变、可序列化且永不抛出异常。
- **依赖注入（构造函数）**：API 接收 `{ transport, zonesAPI?, snapshotOps? }` 对象；除日志配置外，没有任何全局单例。
- **延迟初始化**：`bdclient` 上的 `defineLazy`（首次访问时构建功能，然后替换为实例化后的值），以及 `DatasetsClient` 中访问时缓存的机制。
- **边界验证**：schema 与使用它们的代码放在一起；每个公共入口都会调用 `assertSchema(...)`。不仅输入会被验证，响应也会被验证。

## 横切工具

- `src/utils/polling.ts`：共享的 `pollUntilReady(id, getStatus, opts)` 原语，包含已用时间/截止时间/间隔逻辑和状态回调。
- `src/utils/logger.ts`：进程级配置（`setup(...)`）和命名日志记录器工厂；支持结构化 JSON 和美化输出；除非设置 `verbose`，否则会按阈值过滤。
- `src/utils/misc.ts`：`sleep`、`parseJSON`、`parseResponse`（经 schema 检查）、`dropEmptyKeys`、`maskKey`、`isStrArray`。
- `src/utils/files.ts`：保存结果、生成文件名、将快照下载流式写入磁盘（`routeDownloadStream` 是 undici 流工厂）。
- `src/utils/constants.ts`：端点 URL、重试配置、默认区域。修改 API 接口只需在这一处进行。

## 架构中不太一致的地方

请注意以下几点：它们不是 bug，但如果你期待完全一致，很容易在这里踩坑：

1. **三种略有不同的轮询实现。** `ScrapeJob.wait` 使用共享的 `pollUntilReady` 辅助工具（ready/failed/error 词汇）。`DiscoverJob.wait` 有自己的内联循环（状态词汇：`done` / `error` / `failed`）。`ScraperStudioJob.waitAndFetch` 还有另一种内联循环（没有单独的状态端点，它只会在收到 202 时重试 `fetch`）。不同产品使用不同的服务端契约，但它们从未统一到同一个抽象之下。

2. **`Job` 类接收 HTTP 依赖项的两种方式。** `ScrapeJob` 接收 `SnapshotOperations` 接口（以避免与 `SnapshotAPI` 发生循环导入，代码注释中对此有所说明）。`ScraperStudioJob` 直接接收 `Transport`。`DiscoverJob` 接收 `DiscoverOperations` 接口。三者以三种不同方式解决同一个问题。

3. **`DatasetsClient` 包含 125 个几乎相同的延迟 getter。** 它功能正常，但重复度很高。工厂循环会更紧凑，但当前方式保留了每个属性的静态类型。这个文件需要大量滚动浏览。

4. **有两种名为“数据集”的功能。** 请参阅上面的流程 #5。命名重叠（scrape.* 使用 `/datasets/v3/*`，datasets.* 使用 `/datasets/*`）是底层产品接口遗留的结果。

5. **清理机制采用双重保障。** `Transport` 会注册一个 `beforeExit` 监听器，如果你忘记调用 `close()`，它会发出警告，*同时*还支持 `Symbol.asyncDispose` 用于 `await using`。两条代码路径都有效，并且并存。

6. **浏览器 API 是新增的，功能范围较窄。** `src/api/browser/service.ts` 只构建连接 URL，不涉及 HTTP、任务或结果。其结构与其他功能系列都不同。（应将其视为起点，而非范式。）

## 建议优先阅读的位置（按顺序）

1. `src/client.ts`：查看延迟连接机制，以及所有内容如何挂载到 `bdclient` 上。
2. `src/core/transport.ts`：了解每个 HTTP 调用实际是如何发出的。
3. `src/api/unlocker/request.ts` + `src/api/unlocker/scrape.ts`：最简单的端到端流程。
4. `src/api/scrape/base.ts` + `src/api/scrape/job.ts` + `src/api/scrape/snapshot.ts` + `src/api/scrape/linkedin.ts`：四个简洁文件展示的异步任务流程。
5. `src/utils/polling.ts` + `src/utils/deadline.ts` + `src/models/result.ts`：异步工作的共享词汇。
6. `src/schemas/`：浏览即可，不必细读。了解其模式（每个公共入口点都有 `assertSchema(SomeSchema, input, label)`）；需要特定内容时再回来查看。
7. `src/api/datasets/base.ts`：然后阅读 `client.ts` 和一个平台文件。这就是预采集数据集的完整实现。

在需要添加或更改数据集之前，可跳过 `src/api/datasets/client.ts` 和 `src/api/datasets/platforms/*`，它们高度公式化。
