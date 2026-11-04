# 代码库简介：架构导览

欢迎。本项目是一个 JavaScript/TypeScript SDK，对 Bright Data 的网页爬虫、搜索和代理 API 进行了封装。阅读代码无需了解产品，你只需要关注代码是如何组合在一起的，而本文档正是对此进行介绍。

## 30 秒概览

SDK 由五个同心层组成。数据和控制从用户向外流向网络：

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

用户执行的所有操作都会经过 `bdclient` 外观层，它会延迟构建各个功能接口。所有功能共享同一个用于 HTTP 的 `Transport` 实例、同一个速率限制器和同一组身份验证请求头。

## 主要抽象（你会在各处看到的名词）

**`bdclient`**（`src/client.ts`）。这是用户唯一需要实例化的类。它读取选项和环境变量，构建 `Transport`，并通过一个小型 `defineLazy` 辅助函数，在首次访问时延迟实例化每个子 API。其他所有内容都挂载在该对象的属性上：`client.scrape.linkedin`、`client.search.google`、`client.datasets.amazonProducts`、`client.discover`、`client.scraperStudio`、`client.browser`、`client.listZones()`。

**`Transport`**（`src/core/transport.ts`）。唯一的核心 HTTP 客户端。它使用连接池、keep-alive、重试拦截器、DNS 缓存、可选速率限制器和统一的 `classifyError` 步骤封装 undici；该步骤会将 undici 错误转换为 SDK 自有的类型化错误层级。代码库中所有 HTTP 通信都经过这里，不存在零散的 `fetch` 调用。

**`RateLimiter`**（`src/core/rate-limiter.ts`）。一个带等待队列的令牌桶。`Transport.request` 会在每个出站请求之前调用 `acquire()`。它是可选功能（默认关闭）；你可以通过客户端上的 `rateLimit` 启用。

**功能“路由器”和“服务”**。每个功能系列都有一个由客户端公开的小型分派器类：
- `ScrapeRouter`（`src/api/scrape/router.ts`）：为每个平台爬虫工具各持有一个实例（`linkedin`、`amazon`、`instagram`、`facebook`、`tiktok`、`youtube`、`reddit`、`pinterest`、`digikey`、`chatGPT`、`perplexity`），以及它们共享的一个 `SnapshotAPI`。
- `SearchRouter`：所有 `google / bing / yandex` 方法均委托给同一个 `SearchAPI`。
- `DatasetsClient`：125 个以上的延迟 getter，每个预采集数据集对应一个。
- `DiscoverService`、`ScraperStudioService`、`BrowserService`：独立功能服务。
- `ZonesAPI`：并非路由器，只是一个缓存的 API 包装器。

**`*API` 类**（`src/api/**/*.ts`）。它们是实际构造请求的主力，分为两个系列：
- `RequestAPI`（位于 `src/api/unlocker/`）：由 `ScrapeAPI` 和 `SearchAPI` 继承。同步风格端点（`POST /request`）。数组并发通过 `@supercharge/promise-pool` 完成。基类定义流程，子类覆盖 `getURL` 和 `getMethod`（模板方法模式）。
- `BaseAPI`（位于 `src/api/scrape/`）：由平台爬虫工具（`LinkedinAPI`、`AmazonAPI`……）以及 `SnapshotAPI` 继承。异步、面向数据集的端点（`POST /datasets/v3/trigger|scrape`）。平台子类大多只是使用类型化输入和硬编码的 `datasetId` 封装 `run()` 或 `orchestrate()`。

**`*Job` 类**（`ScrapeJob`、`DiscoverJob`、`ScraperStudioJob`）。用于承载长期运行服务器端任务相关值的句柄。它们具有相同的结构：ID + 触发时间戳、一组 `status()` / `wait()` / `fetch()` / `toResult()` 方法（或等效方法），以及状态和结果的内部缓存。当你触发异步操作但不进行编排时，就会得到这些对象。

**`*Result` 类**（`BaseResult`、`ScrapeResult`、`SearchResult`、`DiscoverResult`）。由 `job.toResult()` 返回的值对象。它们永远不会抛出异常：失败信息会写入带有 `error`、`status` 和计时字段的 `success: false` 结果。它们可通过 `toJSON()` 干净地序列化，也能通过 `save()` 将自身保存到磁盘。

**`BaseDataset`**（`src/api/datasets/base.ts`）。所有 125 个数据集类的抽象父类。每个具体子类实际上只有 3 行（一个硬编码的 `datasetId` + `name`）。基类针对 `/datasets/*` 端点提供 `query()`、`sample()`、`getStatus()`、`download()`，这是经典的仓库模式。

**Zod schema**（`src/schemas/`）。SDK 在边界处执行验证。每个公共方法都以 `assertSchema(...)` 调用开始，将用户输入转换为类型化、规范化的数据。响应解析会针对响应结构 schema 使用 `parseResponse`，从而同时检查服务器契约。`ValidationError` 是这些调用会产生的唯一异常类型。

**错误层级**（`src/utils/errors.ts`）。所有错误均继承自 `BRDError`。具体类型包括：`ValidationError`、`AuthenticationError`、`ZoneError`、`NetworkError` → `NetworkTimeoutError`、`TimeoutError`、`FSError`、`APIError`（携带 `statusCode`）、`DataNotReadyError`。`Transport.classifyError` 和 `throwInvalidStatus` 是决定抛出哪种类型的两个关键节点。

**`Deadline`**（`src/utils/deadline.ts`）。一个使用 `.remaining` 和 `.expired` 封装 `start + budget` 的小型辅助工具。`*Job.toResult()` 风格的方法使用它来对多次重试实施总时间预算，这样竞争条件重试就不会每次都重新开始计算超时。

## 主要流程如何在系统中运行

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

搜索（`client.search.google(q)`）采用相同的管线，只是使用不同的 `*API` 子类（`SearchAPI`），将查询放入特殊的 URL 格式中。

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

原始 `collect*` / `discover*` 方法（例如 `client.scrape.linkedin.collectProfiles`）会更早返回：它们会提供一个由你自行编排的 `ScrapeJob`；如果服务器在同步超时之前完成，则返回同步模式数据。`orchestrate()` 只是一个“通过一次调用完成所有操作”的便捷方法。

### 3) 发现（AI 驱动的搜索）

```
client.discover(query)
  → DiscoverService.search → _trigger → POST /discover → { task_id }
  → new DiscoverJob(task_id, ops)
  → job.toResult()
      → wait() loops GET /discover?task_id=… every 2s until status === 'done'
      → results come back on the status response itself (no separate fetch endpoint)
  → DiscoverResult(success, data=items, totalResults, durationSeconds)
```

请注意它与 ScrapeJob 的细微区别：Discover 的“完成”状态会附带有效载荷，无需单独调用下载。

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

这里有一个值得尽早了解的命名空间冲突：存在**两套不同的“数据集”系统**。`client.scrape.*` 调用 Bright Data 的 `/datasets/v3/` 爬虫端点（你提供输入，它运行采集器）。`client.datasets.*` 调用独立的 `/datasets/` 预采集端点（你查询现有数据集）。除了传输层之外，两者不共享代码。

### 6) 浏览器 API

这是一个特例：完全不进行 HTTP 调用。`client.browser.getConnectUrl({ country })` 返回一个可传给 Puppeteer/Playwright 的 `wss://…` 字符串，随后由后者自行建立实际连接。凭据来自构造函数选项或 `BRIGHTDATA_BROWSERAPI_*` 环境变量，并在首次访问时延迟验证。

## 使用的设计模式

- **外观模式**：`bdclient` 将五个不同的功能系列隐藏在一个友好的对象之后。
- **分层架构**：用户代码 → 功能服务/路由器 → API 类 → Transport → undici。没有任何层会越级向下调用。
- **模板方法**：`RequestAPI`（覆盖：`getURL`、`getMethod`）和 `BaseAPI.run/orchestrate`（子类只需使用自己的 `datasetId` 调用它）。
- **仓库模式**：`BaseDataset` 及约 125 个单行子类，每个子类都是特定远程数据集的仓库。
- **命令 + Future（Job）**：`ScrapeJob`、`DiscoverJob`、`ScraperStudioJob`。触发操作会返回一个句柄；该句柄公开轮询/获取/结果操作。每个类还提供一个便捷的 `toResult()` 方法，将成功和失败包装成 Result 值对象。
- **值对象**：`BaseResult` 层级。不可变、可序列化且永不抛出异常。
- **依赖注入（构造函数）**：API 接收 `{ transport, zonesAPI?, snapshotOps? }` 对象；除日志配置外，不存在全局单例。
- **延迟初始化**：`bdclient` 上的 `defineLazy`（功能在首次访问时构建，随后由已实例化的值替换），以及 `DatasetsClient` 中的按访问缓存。
- **边界验证**：schema 与使用它们的代码放在一起；每个公共入口点都会调用 `assertSchema(...)`。不仅验证输入，也验证响应。

## 横切工具

- `src/utils/polling.ts`：一个共享的 `pollUntilReady(id, getStatus, opts)` 原语，包含已用时间/截止时间/间隔逻辑和状态回调。
- `src/utils/logger.ts`：进程级配置（`setup(...)`）和具名日志记录器工厂；支持结构化 JSON 和美化输出；除非设置 `verbose`，否则会按阈值过滤。
- `src/utils/misc.ts`：`sleep`、`parseJSON`、`parseResponse`（经过 schema 检查）、`dropEmptyKeys`、`maskKey`、`isStrArray`。
- `src/utils/files.ts`：保存结果、生成文件名、将快照下载流式写入磁盘（`routeDownloadStream` 是 undici 流工厂）。
- `src/utils/constants.ts`：端点 URL、重试配置和默认区域。只需在一个位置修改 API 接口。

## 架构中略显不一致的部分

请注意以下几点：它们并非 bug，但如果你预期所有行为完全一致，就可能感到困惑：

1. **三种略有不同的轮询实现。**`ScrapeJob.wait` 使用共享的 `pollUntilReady` 辅助工具（ready/failed/error 词汇）。`DiscoverJob.wait` 有自己的内联循环（状态词汇：`done` / `error` / `failed`）。`ScraperStudioJob.waitAndFetch` 还有另一个内联循环（没有独立的状态端点，只是在收到 202 时重试 `fetch`）。不同产品具有不同的服务器契约，但它们从未统一到同一个抽象之下。

2. **`Job` 类通过两种方式接收 HTTP 依赖。**`ScrapeJob` 接收 `SnapshotOperations` 接口（为避免与 `SnapshotAPI` 发生循环导入，代码注释中记录了这一点）。`ScraperStudioJob` 直接接收 `Transport`。`DiscoverJob` 接收 `DiscoverOperations` 接口。三者以三种不同方式解决同一个问题。

3. <strong>`DatasetsClient` 包含 125 个几乎完全相同的延迟 getter。</strong>它功能正常，但重复程度很高。使用工厂循环会更紧凑，但当前方式保留了每个属性的静态类型。你可能需要在该文件中滚动很久。

4. <strong>两个名为“数据集”的系统。</strong>参见上面的流程 #5。命名重叠（scrape.* 使用 `/datasets/v3/*`，数据集.* 使用 `/datasets/*`）是底层产品接口造成的历史结果。

5. **清理采用双重保障。**`Transport` 注册了一个 `beforeExit` 监听器，在你忘记调用 `close()` 时发出警告，*并且*为 `await using` 支持 `Symbol.asyncDispose`。两条代码路径均有效，并同时存在。

6. **浏览器 API 是新增且功能范围较窄的部分。**`src/api/browser/service.ts` 只构建连接 URL，不涉及 HTTP、任务或结果。它在结构上与其他所有功能系列都不同。（应将其视为起点，而非范式。）

## 建议优先阅读的位置（按顺序）

1. `src/client.ts`：查看延迟装配，以及所有内容如何挂载在 `bdclient` 上。
2. `src/core/transport.ts`：了解每个 HTTP 调用实际如何执行。
3. `src/api/unlocker/request.ts` + `src/api/unlocker/scrape.ts`：最简单的端到端流程。
4. `src/api/scrape/base.ts` + `src/api/scrape/job.ts` + `src/api/scrape/snapshot.ts` + `src/api/scrape/linkedin.ts`：四个简洁文件中的异步任务流程。
5. `src/utils/polling.ts` + `src/utils/deadline.ts` + `src/models/result.ts`：异步工作的共享词汇。
6. `src/schemas/`：浏览即可，不要细读。了解其模式（每个公共入口点都有 `assertSchema(SomeSchema, input, label)`）；需要具体内容时再回来查看。
7. `src/api/datasets/base.ts`：然后阅读 `client.ts` 和一个平台文件。这就是预采集数据集的全部内容。

在需要添加或更改数据集之前，请跳过 `src/api/datasets/client.ts` 和 `src/api/datasets/platforms/*`：它们高度公式化。
