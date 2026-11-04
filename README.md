<img width="1300" height="200" alt="sdk-banner(1)" src="https://github.com/user-attachments/assets/c4a7857e-10dd-420b-947a-ed2ea5825cb8" />

<h3>Bright Data JavaScript SDK，为网页爬虫、网络搜索、数据集等功能提供简单且可扩展的方法。</h3>

## 安装 [![@latest](https://img.shields.io/npm/v/@brightdata/sdk.svg)](https://www.npmjs.com/package/@brightdata/sdk)

```bash
npm install @brightdata/sdk
```

## 快速开始

### 1. [注册](https://www.bright.cn/cp)并获取你的 API 令牌

### 2. 初始化客户端

```javascript
import { bdclient } from '@brightdata/sdk';

const client = new bdclient({
    apiKey: '[your_api_token]', // or set BRIGHTDATA_API_TOKEN env variable
});
```

### 3. 发起你的第一个请求

```javascript
// Scrape a webpage
const html = await client.scrapeUrl('https://example.com');
console.log(html);

// Search the web
const results = await client.search.google('pizza restaurants', { country: 'us' });
console.log(results);
```

完成后别忘了关闭：

```javascript
await client.close();
```

## 功能

- **网页爬虫**：借助反机器人检测绕过和代理支持抓取任意网站
- **搜索引擎结果**：支持批量执行 Google、Bing 和 Yandex 搜索
- **平台爬虫工具**：从 LinkedIn、Amazon、Instagram、TikTok、YouTube、Reddit 等平台采集结构化数据
- **爬虫 API**：爬取任意一个或多个 URL，并获取按页面打包的所有输出格式（Markdown、HTML、文本）
- **发现 API**：由 AI 驱动的网络搜索，提供基于意图的相关性排序
- **爬虫工作室**：触发并获取由 Bright Data 爬虫工作室中自定义抓取工具生成的结果
- **浏览器 API**：用于将 Playwright、Puppeteer 或 Selenium 连接到 Bright Data 云浏览器的 CDP WebSocket URL
- **数据集**：访问覆盖数十个平台的 126 个预构建数据集，支持查询和下载
- **并行处理**：并发处理多个 URL 或查询
- **稳健的错误处理**：带有重试逻辑的类型化错误类
- **区域管理**：自动创建和管理区域
- **多种输出格式**：HTML、JSON、Markdown 和屏幕截图
- **双构建**：同时支持 ESM 和 CommonJS
- **TypeScript**：具有重载签名的完整类型化 API
- **子路径导出**：通过 `@brightdata/sdk/scrapers`、`@brightdata/sdk/search`、`@brightdata/sdk/datasets` 实现可摇树优化的导入

## 用法

### 抓取网站

```javascript
// Single URL — returns HTML string by default
const html = await client.scrapeUrl('https://example.com');

// Multiple URLs (parallel processing)
const results = await client.scrapeUrl([
    'https://example1.com',
    'https://example2.com',
]);

// Get markdown content
const md = await client.scrapeUrl('https://example.com', {
    dataFormat: 'markdown',
});

// Get structured JSON
const data = await client.scrapeUrl('https://example.com', {
    format: 'json',
});

// Take a screenshot
const screenshot = await client.scrapeUrl('https://example.com', {
    dataFormat: 'screenshot',
});

// Full options
const result = await client.scrapeUrl('https://example.com', {
    format: 'raw',            // 'raw' (default) or 'json'
    dataFormat: 'html',       // 'html' (default), 'markdown' (alias: 'md'), 'screenshot'
    country: 'gb',            // two-letter country code
    method: 'GET',            // HTTP method
});
```

### 搜索引擎

```javascript
// Google search
const results = await client.search.google('pizza restaurants');

// Bing search
const results = await client.search.bing('pizza restaurants');

// Yandex search
const results = await client.search.yandex('pizza restaurants');

// Batch search (parallel)
const results = await client.search.google(['pizza', 'sushi', 'tacos']);

// With options
const results = await client.search.google('pizza', {
    country: 'gb',
    format: 'json',
});
```
> <strong>注意：</strong>如果未指定 `country`，请求将通过任意代理位置出口，因此结果的地理位置可能会落在非预期国家/地区。需要一致的本地化结果时，请始终传入 `country`。

### 平台爬虫工具

从热门平台采集结构化数据。每个平台均支持同步采集（`collect*`）和异步编排抓取（触发、轮询、下载）。

```javascript
// LinkedIn profiles
const data = await client.scrape.linkedin.collectProfiles(
    ['https://www.linkedin.com/in/satyanadella/'],
    { format: 'json' },
);

// Amazon products
const data = await client.scrape.amazon.collectProducts(
    ['https://www.amazon.com/dp/B0D77BX8Y4'],
    { format: 'json' },
);

// Instagram profiles
const data = await client.scrape.instagram.collectProfiles(
    ['https://www.instagram.com/natgeo/'],
    { format: 'json' },
);

// TikTok profiles
const data = await client.scrape.tiktok.collectProfiles(
    ['https://www.tiktok.com/@tiktok'],
    { format: 'json' },
);

// YouTube videos
const data = await client.scrape.youtube.collectVideos(
    ['https://www.youtube.com/watch?v=dQw4w9WgXcQ'],
    { format: 'json' },
);

// Reddit posts
const data = await client.scrape.reddit.collectPosts(
    ['https://www.reddit.com/r/technology/top/'],
    { format: 'json' },
);
```

**编排抓取**（异步触发 → 轮询 → 下载）：

```javascript
const result = await client.scrape.linkedin.profiles(
    ['https://www.linkedin.com/in/satyanadella/'],
    { pollInterval: 5000, pollTimeout: 180_000 },
);
console.log(result.data);    // structured data
console.log(result.status);  // 'ready'
console.log(result.rowCount);
```

**可用平台：**`linkedin`、`amazon`、`instagram`、`tiktok`、`youtube`、`reddit`、`facebook`、`pinterest`、`chatGPT`、`digikey`、`perplexity`

### 爬虫 API

爬取一个或多个 URL，并获取按页面打包的所有输出格式（Markdown、HTML、文本）。

```javascript
// Sync — single round-trip
const result = await client.crawler.crawl('https://example.com');
console.log(result.data[0].markdown);

// Batch
const result = await client.crawler.crawl([
    'https://example.com',
    'https://example.com/about',
]);
console.log(`${result.pageCount} pages`);

// Async — trigger, poll, download
const job = await client.crawler.trigger('https://example.com');
const status = await client.crawler.status(job.snapshotId);
const result = await client.crawler.download(job.snapshotId);
```

### 发现 API

由 AI 驱动的网络搜索，提供基于意图的相关性排序。

`discover()` 解析为一个 `DiscoverResult` 包装器（而非裸数组）。项目位于 `result.data`（或其别名 `result.results`）中，且结果可迭代。失败时，`result.success` 为 `false`，`result.error` 包含失败原因，而 `result.data` / `result.results` 保持为空数组，因此迭代永远不会抛出异常。

```javascript
// Basic search
const result = await client.discover('artificial intelligence trends 2026');

if (!result.success) {
    console.error('discover failed:', result.error);
} else {
    console.log(result.results); // [{ link, title, description, relevance_score }, ...]
    for (const item of result) {
        console.log(`[${item.relevance_score}] ${item.title}`);
    }
}

// With intent for semantic ranking
const result = await client.discover('Tesla battery technology', {
    intent: 'recent breakthroughs in EV battery chemistry',
});

// With filtering and localization
const result = await client.discover('sustainable fashion brands', {
    intent: 'eco-friendly clothing companies',
    filterKeywords: ['sustainability', 'eco-friendly', 'organic'],
    country: 'us',
    numResults: 10,
});

// Include full page content
const result = await client.discover('python asyncio tutorial', {
    includeContent: true,
    numResults: 3,
});

// Manual trigger/poll/fetch
const job = await client.discoverTrigger('market research SaaS', {
    intent: 'competitor pricing strategies',
});
await job.wait({ timeout: 60_000 });
const data = await job.fetch();
```

### 爬虫工作室

触发并获取你在[爬虫工作室](https://www.bright.cn/cp/data_collector)中构建的自定义爬虫工具所生成的结果。

```javascript
// Orchestrated — trigger + poll + return results
const results = await client.scraperStudio.run('c_your_collector_id', {
    input: { url: 'https://example.com/product/1' },
});
// results: RunResult[] — one per input with { input, data, error, responseId, elapsedMs }

// Multiple inputs (processed sequentially)
const results = await client.scraperStudio.run('c_your_collector_id', {
    input: [
        { url: 'https://example.com/product/1' },
        { url: 'https://example.com/product/2' },
    ],
});

// Manual control — trigger, then poll yourself
const job = await client.scraperStudio.trigger('c_your_collector_id', {
    url: 'https://example.com/product/1',
});
const data = await job.waitAndFetch();

// Check job status (by job ID from the dashboard)
const status = await client.scraperStudio.status('j_abc123');
console.log(status.status); // 'queued' | 'running' | 'done' | 'failed'
```

### 浏览器 API

构建 CDP WebSocket URL，以将 Playwright、Puppeteer 或 Selenium 连接到 Bright Data 的云浏览器。凭据来自 `browserUsername`/`browserPassword` 选项或 `BRIGHTDATA_BROWSERAPI_USERNAME`/`BRIGHTDATA_BROWSERAPI_PASSWORD` 环境变量。

```javascript
// Get a connection URL
const url = client.browser.getConnectUrl();

// Geo-target the browser with a 2-letter country code
const usUrl = client.browser.getConnectUrl({ country: 'us' });

// Connect with Playwright
import { chromium } from 'playwright';
const browser = await chromium.connectOverCDP(url);
const page = await browser.newPage();
await page.goto('https://example.com');
const html = await page.content();
await browser.close();
```

### 数据集

访问 126 个预构建数据集，以查询和下载结构化数据快照。

```javascript
const ds = client.datasets;

// List all datasets available on your account
const list = await ds.list();

// Get field metadata for a dataset
const meta = await ds.instagramProfiles.getMetadata();
console.log(meta.fields); // [{ name, type, description }, ...]

// Query a dataset (triggers a snapshot)
const snapshotId = await ds.instagramProfiles.query(
    { url: 'https://www.instagram.com/natgeo/' },
    { records_limit: 10 },
);

// Check snapshot status
const status = await ds.instagramProfiles.getStatus(snapshotId);
console.log(status.status); // 'running' | 'ready' | ...

// Download when ready
const rows = await ds.instagramProfiles.download(snapshotId);
```

**可用数据集：**

| 平台 | 数据集 |
|----------|----------|
| LinkedIn | `linkedinProfiles`, `linkedinCompanies` |
| Amazon | `amazonProducts`, `amazonReviews`, `amazonSellers`, `amazonBestSellers`, `amazonProductsSearch`, `amazonProductsGlobal`, `amazonWalmart` |
| Instagram | `instagramProfiles`, `instagramPosts`, `instagramComments`, `instagramReels` |
| TikTok | `tiktokProfiles`, `tiktokPosts`, `tiktokComments`, `tiktokShop` |
| X/Twitter | `xTwitterPosts`, `xTwitterProfiles` |

### 保存结果

```javascript
const data = await client.scrapeUrl('https://example.com');
const filePath = await client.saveResults(data, {
    filename: 'results.json',
    format: 'json',
});
console.log(`Saved to: ${filePath}`);
```

## 配置

### 运行时支持

Node.js（>= 20）是受支持且经过测试的运行时。该 SDK 也可在 Bun 下运行而不会崩溃，但存在一个实际限制：Bun 捆绑的 `undici` 仅提供最基本的 `Agent`（没有 `compose()`、`close()` 或自定义 `dispatch`），且 Bun 的 `request()`/`stream()` 会完全忽略 `dispatcher` 选项，改由 Bun 自己的原生 HTTP 客户端路由。实际上，这意味着请求在 Bun 下仍能成功，但 Transport 的所有调优，连接池、keep-alive、自定义超时、针对 429/500/502/503/504 的自动重试以及 DNS 缓存，均不会生效；系统将改用 Bun 自身的默认设置。SDK 会自动检测缺失的能力（构造过程或 `close()` 永远不会崩溃），而不是直接失败。

### API 令牌

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users?=)获取你的 API 令牌。

<strong>已经通过 CLI 登录？SDK 无需配置即可工作。</strong>如果你已通过 [Bright Data CLI](https://www.npmjs.com/package/@brightdata/cli) 运行 `brightdata login`，SDK 会自动读取存储的凭据。令牌按以下顺序解析：

1. 传递给 `new bdclient({ apiKey })` 的 `apiKey`
2. `BRIGHTDATA_API_TOKEN`（或 `BRIGHTDATA_API_KEY`）环境变量
3. CLI 存储的凭据（`brightdata login`）

如果均未找到，客户端会抛出错误，并给出登录或设置令牌的操作说明。

### 环境变量

```env
BRIGHTDATA_API_TOKEN=your_api_token                  # BRIGHTDATA_API_KEY also accepted
BRIGHTDATA_WEB_UNLOCKER_ZONE=your_web_unlocker_zone  # Optional
BRIGHTDATA_SERP_ZONE=your_serp_zone                  # Optional
BRIGHTDATA_BROWSERAPI_USERNAME=your_browser_username # Optional, for Browser API
BRIGHTDATA_BROWSERAPI_PASSWORD=your_browser_password # Optional, for Browser API
BRIGHTDATA_VERBOSE=1                                  # Optional, enable verbose logging
```
> <strong>提示：</strong>使用 `node --env-file=.env` 从 `.env` 文件加载这些变量时，请注意 shell 中已设置的变量优先于文件中的变量。

### 客户端选项

```javascript
const client = new bdclient({
    apiKey: 'string',           // API token (or use BRIGHTDATA_API_TOKEN env var)
    timeout: 120000,            // Request timeout in ms (1000–300000)
    autoCreateZones: true,      // Auto-create zones if they don't exist
    webUnlockerZone: 'string',  // Custom web unlocker zone name
    serpZone: 'string',         // Custom SERP zone name
    logLevel: 'INFO',           // 'DEBUG' | 'INFO' | 'WARNING' | 'ERROR' | 'CRITICAL'
    structuredLogging: true,    // Use structured JSON logging
    verbose: false,             // Enable verbose logging
    rateLimit: 0,               // Max requests per period (0 = unlimited)
    ratePeriod: 1000,           // Rate limit period in ms
});
```

### 资源清理

客户端会维护 HTTP 连接。完成后请始终关闭：

```javascript
await client.close();

// Or use Symbol.asyncDispose (TypeScript 5.2+)
await using client = new bdclient();
```

### 常量

| 常量               | 默认值  | 说明                    |
| ---------------------- | -------- | ------------------------------ |
| `DEFAULT_CONCURRENCY`  | `10`     | 最大并行任务数             |
| `DEFAULT_TIMEOUT`      | `120000` | 请求超时（毫秒） |
| `MAX_RETRIES`          | `3`      | 失败时的重试次数      |
| `RETRY_BACKOFF_FACTOR` | `1.5`    | 指数退避乘数 |

### 区域管理

```javascript
const zones = await client.listZones();
console.log(`Found ${zones.length} zones`);
```

## 子路径导出

如需摇树优化或仅导入所需内容：

```javascript
import { ScrapeRouter, LinkedinAPI } from '@brightdata/sdk/scrapers';
import { SearchRouter } from '@brightdata/sdk/search';
import { DatasetsClient, BaseDataset } from '@brightdata/sdk/datasets';
```

## 错误处理

SDK 导出扩展自 `BRDError` 的类型化错误类：

```javascript
import { bdclient, ValidationError, AuthenticationError, BRDError } from '@brightdata/sdk';

try {
    const result = await client.scrapeUrl('https://example.com');
} catch (error) {
    if (error instanceof ValidationError) {
        console.error('Invalid input:', error.message);
    } else if (error instanceof AuthenticationError) {
        console.error('Auth failed:', error.message);
    } else if (error instanceof BRDError) {
        console.error('SDK error:', error.message);
    }
}
```

**错误类型：**`ValidationError`、`AuthenticationError`、`ZoneError`、`NetworkError`、`NetworkTimeoutError`、`TimeoutError`、`APIError`、`DataNotReadyError`、`FSError`

## 故障排除

### Windows 和企业网络

| 问题 | 解决方法 |
|---|---|
| `npm.ps1 cannot be loaded because running scripts is disabled`（PowerShell） | 使用 **cmd** 代替 PowerShell，或运行 `npm.cmd <command>`，或运行 `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`。如果设置被组策略覆盖，请联系你的 IT 团队。 |
| `npm install` 期间出现 `SELF_SIGNED_CERT_IN_CHAIN` | 你的网络使用了 SSL 检查。将 npm 指向企业根证书：`npm config set cafile "C:\path\to\corporate-root.cer"` |
| 运行时证书错误 | 设置环境变量 `NODE_EXTRA_CA_CERTS=C:\path\to\corporate-root.cer`，让 Node 信任你的企业证书。 |

### AuthenticationError：API 密钥无效

如果遇到此错误，请按以下顺序检查：

1. **验证令牌本身**是否可在 SDK 外正常使用：
   ```bash
   curl -H "Authorization: Bearer YOUR_TOKEN" https://api.brightdata.com/zone/get_active_zones
   ```
   如果失败，请在[控制面板](https://www.bright.cn/cp/setting/users)中生成一个具有 **admin** 权限的新 API 密钥。请注意，SDK 需要账户级 API 令牌，而不是区域密码。
2. <strong>检查 shell 覆盖项。</strong>shell 中设置的环境变量优先于通过 `node --env-file=.env` 加载的 `.env` 文件。运行 `set BRIGHTDATA`（Windows）或 `env | grep BRIGHTDATA`（macOS/Linux），并清除所有残留值。
3. <strong>检查你的 `.env` 文件：</strong>不要使用引号，`=` 两侧不要有空格，不要有尾随空白，并确保令牌完整复制（包括连字符）。建议使用 LF 换行符。
4. **输出实际加载的内容：**
   ```js
   console.log(JSON.stringify(process.env.BRIGHTDATA_API_TOKEN));
   ```
   `\r` 等隐藏字符将在输出中显示。

## 支持

如有任何问题，请联系 [Bright Data 支持团队](https://www.bright.cn/contact)，或在此仓库中提交 issue。

## 许可证

本项目采用 [MIT 许可证](https://opensource.org/licenses/MIT)。
