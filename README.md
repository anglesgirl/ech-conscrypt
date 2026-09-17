# ECH Android 集成手册（本地代理 + Conscrypt + WebView 原生登录）

> 本仓库是 **AI 参考手册**：当 App 需要在中国大陆无代理环境下访问被 SNI 阻断的 HTTPS 站点
> 并完成登录/OAuth 时，先读本文件，按方案实施，别再踩一遍坑。
>
> 实战来源：animeko（Android，Kotlin Multiplatform）Bangumi OAuth 登录 + ECH 全流程踩坑记录。
> 适用于：任何"目标站点被墙、App 要内置访问能力并完成表单/OAuth 登录"的场景。

---

## 0. 一句话结论

**WebView 不要直接访问被墙域名，也不要 JS 劫持表单；起一个进程内本地代理（Kotlin ServerSocket），
让 WebView 打开 `http://127.0.0.1:8888/...`，代理用 Conscrypt+ECH 转发到真实域名。
页面、验证码、表单提交全部由浏览器原生处理，零劫持、零冲突。**

- **Conscrypt 版（推荐）**：代理跑在 App 进程内（线程池），不会被系统杀，快/轻/稳
- **Go 版（旧）**：方法完全一样，但 Go 是外挂独立进程，Android 会回收进程 → 不稳定，弃用

---

## 1. 背景：为什么必须这样做

### 1.1 问题链

| 尝试 | 结果 | 原因 |
|---|---|---|
| WebView 直接打开被墙站点 | 连接被重置 | WebView 自带 TLS 栈无 ECH，SNI 明文被墙 RST |
| 系统浏览器 / Custom Tab 打开 | 打不开 | 其他用户的手机没有代理、没有 ECH、没有目标站登录态 |
| `shouldInterceptRequest` 拦截 GET + 自己发 | GET 能通 | 但 **拿不到 POST body**（Android WebView API 限制） |
| JS 劫持表单 POST 走 OkHttp ECH | 验证码永远错误 | 目标站 JS 在提交时会刷新服务器验证码，劫持提交 = 用户输入的是旧验证码 |
| 原生 Activity 拼表单（账号/密码/验证码输入框） | 失败 | 论坛表单动态字段多（formhash/验证码/提交按钮），手拼必然出错；验证码 GIF 解码不可靠 |
| **本地代理 + WebView 原生（本方案）** | **成功** | WebView 把 127.0.0.1 当本地站点，一切原生，代理只做 ECH 传输 |

### 1.2 为什么不直接给 WebView 设代理

- Android WebView **没有 setProxy API**（平台限制）
- 唯一"让 WebView 走代理"的干净办法：**WebView 直接导航到本地代理的 URL**
- 本地代理当"网站"用：页面里的相对路径（表单 action、验证码图片等）自动命中本地代理

---

## 2. 架构（Conscrypt 进程内本地代理）

```
用户点击登录（App 拿到 OAuth 授权 URL，指向被墙目标站点）
        │
        ▼
App 把 URL 的 host 换成 127.0.0.1:8888，WebView 加载（保留路径和 query）
        │
        ▼
进程内本地代理（Kotlin ServerSocket @ 127.0.0.1:8888，App 进程内线程，系统杀不掉）
        │  用 Conscrypt + ECH 转发（SNI 隐藏，绕过封锁）
        ▼
目标站点（原 host + 原路径）
        │
        ├─ 登录页 HTML（相对路径自动回 127.0.0.1）→ WebView 原生渲染（含验证码图片）
        ├─ 表单 POST（浏览器原生提交）→ 代理 → ECH → 目标站点
        ├─ 302 → Location 重写为 127.0.0.1（目标站内部）或原样放行（回调后端，WebView 原生跳转）
        └─ Set-Cookie → 代理内 cookie store（目标站域），请求自动带回
```

代理每个请求的处理步骤：

```kotlin
// 1. 解析 HTTP 请求行/headers/body（Content-Length）
// 2. 用 OkHttp(Conscrypt ECH) 转发到 https://<目标host> + path
// 3. 转发关键 headers；Referer/Origin 重写为真实域名（见坑 #10）
// 4. Set-Cookie 存入内存 cookieStore；请求时带回 Cookie 头
// 5. 302 Location：目标站内部 → 重写为 127.0.0.1；回调后端域 → 原样放行
// 6. HTML 响应：目标站绝对链接 → 替换为 127.0.0.1；相对路径不动
// 7. 返回 HTTP/1.1 + Connection: close（最简，避免 keep-alive 复杂度）
```

---

## 3. 登录页编写方法（一步步）

1. **拿到 OAuth 授权 URL**（一般由 App 自己的后端生成，App 只管打开）
2. **启动本地代理**（失败则提示无法登录，不要回落明文直连）
3. **WebView 加载** `url.replace("https://<目标host>", "http://127.0.0.1:8888")`（保留路径和 query）
4. **什么都不用劫持**：
   - 验证码图片（相对路径）→ 自动走本地代理 → ECH → 正常显示
   - 登录表单（相对 action）→ 浏览器原生提交 → 本地代理收到完整 POST body（含 formhash/账号/密码/验证码/提交按钮等）→ ECH 转发
   - 页面 JS（含"提交时刷新验证码"之类的逻辑）→ 浏览器原生执行，与服务器状态天然一致，无冲突
5. **302 处理**（本地代理里）：
   - `Location: /xxx`（相对）→ 不动，WebView 按 127.0.0.1 解析
   - `Location: https://<目标host>/xxx` → 重写为 `http://127.0.0.1:8888/xxx`
   - `Location: <回调后端域>/...` → **原样放行**，WebView 原生跳转完成绑定
6. **登录成功判据**：拿到 302 且 Location 不含 "login"（参考 Han1meViewer）

---

## 4. 踩坑清单（按时间线，AI 必读）

| # | 坑 | 现象 | 解法 |
|---|---|---|---|
| 1 | OkHttp 自动跟随 302 吞掉 Location | 登录成功但 App 拿不到跳转地址 | POST 用 `followRedirects(false)`，手动读 `Location` |
| 2 | 没有 CookieJar | 登录态丢失，页面反复跳回登录 | 统一 cookie 管理（本方案：代理内 cookie store） |
| 3 | `shouldInterceptRequest` 失败返回 null | WebView 明文直连被墙 → "连接被重置" | **fail-closed**：失败返回 502 错误页，绝不返回 null |
| 4 | 用 `indexOf(域名)` 匹配整个 URL | 页面里的统计/第三方参数（如 `dl=<目标站>...`）被误判成目标站请求，每次交互触发 DoH ECH → 卡顿 | **用 hostname 精确匹配**（Kotlin + JS 双重校验） |
| 5 | 登录 POST 缺 Referer 头 | 目标站防 CSRF 直接返回登录页 | 转发时带 `Referer: <登录页URL>` + `Origin: <目标站origin>` |
| 6 | 页面 `document.write` 覆盖注入的劫持脚本 | 劫持失效 | 避免依赖 document.write 之后的注入（本方案无注入，天然免疫） |
| 7 | FormData 不含提交按钮字段 | 服务器返回登录页（缺 `submit=登录` 之类的按钮字段） | 表单提交按钮 `<button name=value>` 不在 FormData 里，需补（本方案浏览器原生提交自带，无需处理） |
| 8 | 目标站提交时刷新验证码 | JS 劫持提交：用户输入的验证码是旧的 → 服务器报"验证码错误" | 不要劫持！让浏览器原生提交（本方案） |
| 9 | 原生 Activity 拼表单 + GIF 验证码 | 图片区空白（GIF 解码不可靠）；动态字段拼不对 | 放弃原生表单，用 WebView 原生渲染 |
| 10 | **Referer/Origin 是 127.0.0.1** | 目标站报"来路不正确或验证字串不符"（CSRF 校验） | 代理转发时把 Referer/Origin 中的 `http://127.0.0.1:8888` **重写为真实域名的 https** |
| 11 | 验证码图片/session 一致性 | 图片 GET 与 POST 的 cookie 不一致 → 验证码错 | 图片 GET 和 POST 必须带**同一个 cookie store** 的 cookie（本方案代理内统一） |
| 12 | Kotlin 编译：deprecation 当 error | `toHttpUrl` 未 import 编译失败 | 必须显式 `import okhttp3.HttpUrl.Companion.toHttpUrl` 后写 `url.toHttpUrl()` |

---

## 5. 其他要点

- **DoH**：用 IP 直连 DoH（`https://<ip>/dns-query`）防 DoH 域名被污染；DoH 网关域名用内置 IP 解析防死循环
- **ECH fallback**：目标站无 ECH 记录时，fallback 到 Cloudflare 系域名的通用 ECH（Cloudflare 托管域名都可用）
- **OAuth 后端模式（推荐）**：App 从自己的后端申请授权链接 → WebView/浏览器完成授权 → 302 回 App 后端绑定 → App 轮询拿 token。**App 只碰自己的后端（国内可直连），目标被墙站点只走 ECH 代理**，App 自己完全不直连被墙域名
- **编译丢 GitHub Actions**：被墙站点在国内无法直接抓包/诊断，抓取分析必须在境外 runner 上做

---

## 6. Go 版（同方法，弃用原因）

- Go 版：`gomobile bind` 起**独立进程**的 ECH 代理，WebView 同样打开 `http://127.0.0.1:8888/login`
- **方法完全一样**（本地代理 + WebView 原生），只是代理实现是 Go 独立进程
- **弃用**：独立进程会被 Android 系统回收（OOM/后台清理）→ 代理崩溃 → 登录中断
- **Conscrypt 版优势**：进程内线程，随 App 生命周期，系统杀不掉；无 gomobile AAR；更快更轻

---

## 7. 快速自查清单（AI 改完代码后对照）

- [ ] 代理只监听 127.0.0.1，不对外
- [ ] 转发用 Conscrypt + ECH（OkHttp sslSocketFactory + 自定义 DNS）
- [ ] Referer/Origin 已重写为真实域名（坑 #10）
- [ ] cookie 统一在代理内维护，图片 GET 与 POST 同 cookie（坑 #11）
- [ ] 302 回调后端域原样放行
- [ ] HTML 绝对链接已替换为 127.0.0.1，相对路径不动
- [ ] 不做任何 JS 劫持/表单构造（坑 #7 #8）
- [ ] 失败 fail-closed，绝不回落明文（坑 #3）
- [ ] hostname 精确匹配，不 indexOf 整 URL（坑 #4）
