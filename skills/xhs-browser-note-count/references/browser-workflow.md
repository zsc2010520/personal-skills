# 浏览器主页与分页流程

此流程已在 2026-10-12 对研旅桑园主页验证：首屏、`user_posted` 分页响应包含标题、类型和时间；本次结果为图文 147、视频 29、共 176 条。此数字仅是当时的验证记录，后续运行必须重新读取实时数据。

## 1. 连接浏览器插件

使用 Browser 插件及其 Node REPL 运行环境。优先遵循当次插件提供的引导与完整文档，发现本机当前插件版本路径，不固定版本号、浏览器 ID 或标签 ID。不要用通用 Playwright 包连接其他浏览器代替用户指定的内置浏览器。

本次可用引导形式如下，`browserClientPath` 必须来自已发现的当前插件安装路径：

```js
var { setupBrowserRuntime } = await import(browserClientPath);
var agent = await setupBrowserRuntime({ globals: globalThis });
var browser = await agent.browsers.get('iab');
nodeRepl.write(await browser.documentation());
nodeRepl.write(await browser.tabs.list());
```

`setupBrowserRuntime` 返回 `agent`，必须接收返回值。后续保持这些绑定。复用与目标网址匹配的标签；无匹配标签才用 `browser.tabs.new()`，再 `tab.goto(csvUrl)`。读取 `tab.playwright.domSnapshot()` 或文档提供的 `tab.ax` 状态，核验主页昵称。登录失效时交由用户登录后继续。

工具接口会随插件变化；只有阅读当次有效文档后才调用对应方法。浏览器当前标签应通过当前 `tabs.list()` 的 ID 获取。

## 2. 保存首屏数据

主页曾在 DOM 内嵌脚本中提供 `window.__INITIAL_STATE__`。可用只读 DOM 提取对应 script 文本，再在 REPL 中解析数据，避免输出整个状态（可能包含登录信息）。

```js
var raw = await tab.playwright.evaluate(() =>
  Array.from(document.scripts)
    .map(s => s.textContent || '')
    .find(s => s.includes('window.__INITIAL_STATE__'))
);
```

已验证字段：

- `state.user.userPageData.basicInfo.nickname`：网页昵称。
- `state.user.notes`：数组的数组，展开后每项有 `id`、`noteCard`。
- `noteCard`：`time`、`type`、`displayTitle`、`noteId`、`interactInfo.sticky`。

脚本赋值不是保证合法的 JSON。先检查包装格式；解析时只处理字符串之外的裸 `undefined` 等已观察到的值，不能全局替换标题中的文字，也不能执行整段网页脚本。结构变化时检查实际字段，不能沿用旧字段臆测。

把首屏规范化为 `{id,type,title,time,sticky}`，保存到本次账号记录中。

## 3. 监听页面真实请求

使用标签的 `cdp` 能力，先读取其文档，然后启用 Network：

```js
var cdp = await tab.capabilities.get('cdp');
nodeRepl.write(await cdp.documentation());
await cdp.send('Network.enable');
```

已观察到的接口路径是：

`https://edith.xiaohongshu.com/api/sns/web/v1/user_posted`

参数包括 `num=30`、`cursor`、`user_id`，以及页面自行提供的图像格式和来源参数。以当次页面实际请求为准，不使用附件中的旧 Cookie 或签名重放请求，不逆向或伪造签名。

响应结构：

```json
{
  "data": {
    "cursor": "下一页游标",
    "has_more": true,
    "notes": [
      {"note_id":"帖子ID","type":"normal","display_title":"标题","time":1791783170000}
    ]
  }
}
```

确认请求 `user_id` 是当前主页用户 ID；响应账号字段（若有）也要一致。每页保留请求游标、返回游标、条数与日期范围，便于验证连续性。

## 4. 滚动并收集每页

1. 滚动前 `cdp.readEvents()` 获取当前事件游标。
2. 用插件文档提供的正常页面交互滚动。例如本次 `tab.ax.scroll([900,700], 'down', 10)` 可触发分页；坐标必须根据当次视口/截图确认落在内容滚动区域，不固定沿用这个示例。
3. 读取最新页面状态，再从事件游标读取 `Network.responseReceived`，持续处理 `hasMore`，更新到返回事件游标。
4. 过滤接口路径匹配的真实 XHR/Fetch 响应，**排除 `type === 'Preflight'`**。OPTIONS 预检也可能返回 200，但不含笔记正文。
5. 等待该 `requestId` 的 `Network.loadingFinished` 后调用 `Network.getResponseBody`。正文未就绪时保留该 ID 继续等，不能跳过该页。`base64Encoded` 为真时先解码，再解析 JSON。
6. 只保存账号、帖子 ID、标题、类型、时间、置顶状态和分页审计信息。按帖子 ID 合并同一条传输记录，不合并不同帖子。
7. 一次滚动可能自动返回多页。全部处理，核对请求游标连接上首屏末 ID 或上一响应游标。响应出现乱序时按游标链恢复顺序，不能按事件返回顺序直接认为完整。
8. 页面加载是异步的：某一次事件读取没看到新 XHR，不等于没有更多内容。重新读取页面加载状态及后续事件；在页面状态没有变化时不要盲目循环滚动。

当 `readEvents.truncated=true` 时，检查是否丢失未处理的分页响应；已完整保存的旧事件被淘汰不影响结果。无法证明游标链连续时重读该账号，不用缺页数据发布完整结果。

## 5. 月份边界及落盘

把时间戳按 `Asia/Shanghai` 转换。置顶内容单独判定月份；核验非置顶内容排序后继续读到上月。记录第一条上月非置顶内容的 ID、标题和北京时间作为边界证据。

每账号完成后立即写明细文件，并从实际文件复核序号末项、图文数、视频数、总数和所有时间是否属于目标月。批量汇总沿用入口文件中的输出约定。

## 6. 超时恢复

先区分 `tabs.list()`、导航、标题读取、DOM/AX读取、滚动、响应正文哪个操作失败。能列出标签和标题不代表页面读取已正常。

- 阅读插件的 `browser-troubleshooting`，按实际错误选择文档支持的替代读取方式。
- 使用已有 Browser 绑定；仅当明确断连或运行环境因超时重置时重新引导。标签消失时重新取得当前标签或在同一内置浏览器中新建。
- 限制单次故障重试，保存已经验证的账号结果，避免反复相同超时、重置和重复打开页面。
- 本次关闭重开标签未解决持续读取超时；一次后续 Codex 重启后，DOM读取、AX滚动及 Network响应读取恢复。不能把重启当作保证有效的修复，也不能因用户能手动滚动就宣称工具连接恢复。
- 只有实际读取主页和一页新增接口数据后，才报告已恢复；若仍失败，说明具体失败阶段，不能输出不完整总数。
