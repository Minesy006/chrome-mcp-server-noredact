# chrome-mcp-server (no-redact) 相对原版的区别

- 当前版本：**1.0.4**（2026-09-27）
- 安装路径：`/Users/minesy/Downloads/chrome-mcp-server-noredact`
- 扩展名：`chrome-mcp-server (no-redact)`
- 上游：hangwin/mcp-chrome **1.0.0**
- 扩展 ID 未变（`manifest.key` 保留），可在原 unpacked 目录上直接 Reload

定位变化：

- 原版：给 AI 用的浏览器助手——脱敏、截断、只打当前标签、抓包当摘要。
- 本版：已登录 Chrome 的操作员 + 凭据/流量提取器——明文、完整头、cookie jar、可复制 curl。
- 浏览器点选 / 导航 / 填表未改。差的是渗透相关的数据通道。
- 不做 Burp Repeater / Intruder。Chrome 出登录态和流量，变异和批量仍走 curl 或 Burp。

---

## 1. 敏感信息不脱敏

| | 原版 | 本版 |
|---|---|---|
| `chrome_javascript` 输出 | cookie / JWT / Bearer / base64 / hex / password / query 打成 `<redacted>` | 原样返回 |
| 对象里的 `token`、`authorization` 等 key | 整段挡住 | 不挡 |

实现：`sanitizeText` / `isSensitiveKey` 关闭；`sanitizeAndLimitOutput` 只做体积/深度限制，不再打码。成功响应带 `redacted: false`、`noRedactionPatch: "2026-09-27"`。

---

## 2. Cookie（新工具）

原版没有 `chrome_cookies`，也没有 `cookies` 权限。HttpOnly（如小米 `serviceToken`）页面 JS 读不到。

本版：

- 权限：`cookies`
- 工具：`chrome_cookies`
- action：`getAll` / `get` / `set` / `remove`
- 含 HttpOnly
- `format`：`json`（默认）/ `header`（Cookie 请求头）/ `netscape`（curl cookie 文件）
- `getAll` 可按 `url` / `domain` / `name` 过滤；省略时用当前标签 URL

---

## 3. 发请求 `chrome_network_request`

| | 原版 | 本版 |
|---|---|---|
| 从哪发 | 只打**当前激活标签**，页面 `fetch` | 可指定 `tabId` |
| 怎么发 | 页面上下文，吃 CORS，不能随便改 Cookie | **扩展后台 `fetch` + cookie jar**（含 HttpOnly），少 CORS |
| Origin / Referer | 页面自动带 | 未指定时从目标 tab URL 补 |
| 失败 | 直接失败 | 后台失败再回退页面 fetch |
| `formData` | 页面构造 | 仍走页面（需要 DOM FormData） |

后台请求会标明 `from: "background"`、`cookieAttached`、`tabId`。

---

## 4. 抓包 `chrome_network_capture`

| | 原版 | 本版 |
|---|---|---|
| 条数上限 | 100 | 2000，可用 `maxRequests` 调整 |
| 请求头 | 抽「公共头」，Cookie 经常没有 | 完整 `requestHeaders` / `responseHeaders`；`extraHeaders` 拿 Cookie / Set-Cookie |
| 请求体 | 原始 bytes 基本不可用 | UTF-8 解码（上限 512KB） |
| 过滤 | `includeStatic:false` 把 **HTML / `text/*` 当静态丢掉**（实测可 0 条） | 文档、XHR/fetch、JSON/API **一律留**；false 只丢图/字体/css/js |
| 筛选 | 无 | `urlFilter`（子串或 `/regex/flags`） |
| 导出 | 没有可跑命令 | 每条带现成 **`curl`**（方法、URL、全头含 Cookie、body） |

`needResponseBody:true` 仍走 Debugger API 拉响应体，可能和 DevTools 冲突。webRequest 默认模式没有响应体。

debugger 模式对 HTML 仍可能不拉 response body（body 采集偏 API MIME）。不影响请求行、头、curl。

---

## 5. 执行 JS `chrome_javascript`

| | 原版 | 本版 |
|---|---|---|
| 输出上限 | 50KB，深度 6，字符串 10KB | **512KB**，深度 12，字符串 200KB，数组/对象键 2000 |
| DevTools 占用 debugger | 静默掉 **isolated world**，页面闭包/token 读不到，还不说明 | 优先 **MAIN world**；isolated 只作最后手段，并写入 `warnings` |
| 抓包占用 debugger（`needResponseBody`） | 同样可能把 JS 搞哑巴 | **复用同一条 CDP**，继续 `Runtime.evaluate` 页面上下文 |
| 失败提示 | 几乎没有 | `warnings` 写明降级原因 |

`maxOutputBytes` 默认 524288，仍可按调用覆盖。

---

## 6. MCP 连接（bridge，不是扩展 UI）

原版：`127.0.0.1:12306` 单会话。客户端带着旧 `mcp-session-id` 再 `initialize` → `Invalid MCP request`，整条通道死。

本版（`mcp-chrome-bridge`）：再来一次 `initialize` 会丢掉旧 transport、开新会话。不用杀进程、不用新开对话。

未做：真正的多会话。Claude Code 和另一条客户端同时占 12306 仍会互抢。

补丁文件：

- `/opt/homebrew/lib/node_modules/mcp-chrome-bridge/dist/server/index.js`
- schema：`/opt/homebrew/lib/node_modules/mcp-chrome-bridge/node_modules/chrome-mcp-shared/dist/index.js` 与 `index.mjs`

`npm update` / 重装 bridge 可能覆盖这些文件。

---

## 7. 权限与包装

相对原版增加：

- `permissions`：`cookies`
- 扩展名：`chrome-mcp-server (no-redact)`
- `version`：`1.0.4`
- `background.js` 文件头带 pentest patch 标记

未改：host_permissions `<all_urls>`、其余原有权限、popup / sidepanel / content scripts。

---

## 8. 没改的（和原版一样）

- 点、填、键盘、截图、读页面、多标签、dialog、download
- 没有拦截改包、没有 Intruder、没有批量 IDOR
- 没有无痕 / 第二 cookie store（双账号仍需另开窗口或 profile）
- GIF / performance / 书签 / userscript 暴露
- 出口代理 / SOCKS 轮换（Chrome 进程级，不是工具参数）

---

## 9. 工具对照（渗透相关）

| 工具 | 原版 | 本版 |
|---|---|---|
| `chrome_javascript` | 脱敏 + 50KB 截断；debugger 忙则 isolated | 不脱敏；512KB；MAIN / 复用 capture CDP |
| `chrome_network_request` | 当前 tab 页面 fetch | `tabId` + 后台 cookie jar |
| `chrome_network_capture` | 100 条、砍头、滤掉 HTML、无 curl | 2000、全头、保留 HTML/XHR、每条 curl |
| `chrome_cookies` | 无 | 有，含 HttpOnly |
| 其余浏览器工具 | 有 | 行为同原版 |

---

## 10. 推荐工作流

1. 用户在真实 Chrome 登录目标。
2. `chrome_cookies` dump 或 `chrome_network_capture` 抓一条带 Cookie 的请求。
3. 复制 `curl`，或用 `chrome_network_request` 带登录态再打一次。
4. 批量 IDOR / 改包 / Intruder → Burp 或 curl，不要堆进 MCP。
5. 单会话互踢、fido 一探就吊销、logout 炸弹：操作纪律，不是功能开关。

---

## 11. 版本记录

| 版本 | 内容 |
|---|---|
| 1.0.0 | 上游原版 |
| 1.0.1 | 关闭 JS 脱敏 |
| 1.0.2 | `chrome_cookies`；抓包 2000 / 全头 / body 解码 / urlFilter；MCP initialize 恢复旧会话 |
| 1.0.3 | `network_request` 支持 tabId + 后台带 cookie；抓包输出 curl；JS 截断放宽；debugger 降级走 MAIN |
| 1.0.4 | 默认抓包保留 HTML/XHR/JSON；JS 复用 capture 占用的 debugger 会话 |

---

## 12. Reload

`chrome://extensions` → **chrome-mcp-server (no-redact)** → 重新加载。

若 Chrome 弹出 cookies 权限，点允许。

MCP 工具描述刷新：重连 `streamable-mcp-server`（`http://127.0.0.1:12306/mcp`）或新开一轮对话。
