# ECH Android 集成手册（本地代理 + Conscrypt + WebView 原生登录）

> 本仓库是 **AI 实施手册**：当 App 需要在中国大陆无代理环境下访问被 SNI 阻断的 HTTPS 站点
> 并完成登录/OAuth 时，**按本文件从零到一实施**，别再踩一遍坑。
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

| 尝试 | 结果 | 原因 |
|---|---|---|
| WebView 直接打开被墙站点 | 连接被重置 | WebView 自带 TLS 栈无 ECH，SNI 明文被墙 RST |
| 系统浏览器 / Custom Tab 打开 | 打不开 | 其他用户的手机没有代理、没有 ECH、没有目标站登录态 |
| `shouldInterceptRequest` 拦截 GET + 自己发 | GET 能通 | 但 **拿不到 POST body**（Android WebView API 限制） |
| JS 劫持表单 POST 走 OkHttp ECH | 验证码永远错误 | 目标站 JS 在提交时会刷新服务器验证码，劫持提交 = 用户输入的是旧验证码 |
| 原生 Activity 拼表单（账号/密码/验证码输入框） | 失败 | 论坛表单动态字段多（formhash/验证码/提交按钮），手拼必然出错；验证码 GIF 解码不可靠 |
| **本地代理 + WebView 原生（本方案）** | **成功** | WebView 把 127.0.0.1 当本地站点，一切原生，代理只做 ECH 传输 |

关键平台限制：Android WebView **没有 setProxy API**。唯一"让 WebView 走代理"的干净办法是
**WebView 直接导航到本地代理的 URL**，代理当"网站"用——页面里的相对路径（表单 action、
验证码图片、JS、CSS）自动命中本地代理。

---

## 2. 从零到一实施步骤（照做即可）

### 2.0 依赖与前置

```kotlin
// build.gradle.kts (app module)
dependencies {
    implementation("org.conscrypt:conscrypt-android:2.5.2")   // Conscrypt 提供 ECH
    implementation("com.squareup.okhttp3:okhttp:4.12.0")      // 转发 HTTP 客户端
}
// AndroidManifest.xml：INTERNET 权限（正常 App 都有）
<uses-permission android:name="android.permission.INTERNET" />
```

### 2.1 组件清单（6 个 Kotlin 文件，全部可独立拷贝）

| 文件 | 职责 | 要点 |
|---|---|---|
| `ConscryptEch.kt` | 注册 Conscrypt Provider + ECH 强制策略 | ECH 策略 `REQUIRED`（无配置就报错，绝不空转）；通过自定义 TrustManager 的 `getNetworkSecurityPolicy()` 注入（Conscrypt 反射读取） |
| `EchDoh.kt` | DoH 取 A 记录 + ECH 配置（dns-json） | 带缓存；多节点池轮询；**网关域名用内置 IP 直连**避免死循环；目标站 CNAME 到 Cloudflare 时可用内置 ECH 兜底 |
| `EchSocketFactory.kt` | 为 OkHttp 提供 ECH SocketFactory | 每次新 TLS 连接时向回调取 ECH 配置，设置到 SSLParameters |
| `EchHttp.kt` | OkHttp 客户端（Conscrypt ECH） | **GET 跟重定向 / POST 不跟**（手动读 Location）；CookieJar 与 WebView CookieManager 双向同步 |
| `EchLocalProxy.kt` | 进程内本地代理（ServerSocket @127.0.0.1:8888） | 解析 HTTP 请求→ECH 转发→重写 Referer/302/HTML 链接→代理内 cookie store |
| `EchInternalBrowserActivity.kt` | WebView 登录页 | 打开 127.0.0.1 地址；拦截 `ani://` 自定义 scheme 交给 App |

### 2.2 ConscryptEch.kt（注册 + ECH 策略）

```kotlin
internal object ConscryptEch {
    fun ensureProvider() {
        if (Security.getProvider("Conscrypt") == null) {
            Security.insertProviderAt(Conscrypt.newProvider(), 1)  // 必须插到第一位
        }
    }
    fun systemTrustManager(): X509TrustManager {
        val f = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm())
        f.init(null as java.security.KeyStore?)
        return f.trustManagers.filterIsInstance<X509TrustManager>().first()
    }
    // ECH 策略：只对目标 host REQUIRED，其余 DISABLED（避免影响 App 其他请求）
    class BgmPolicy : NetworkSecurityPolicy {
        override fun getDomainEncryptionMode(hostname: String): DomainEncryptionMode =
            if (isTargetHost(hostname)) DomainEncryptionMode.REQUIRED
            else DomainEncryptionMode.DISABLED
        // isCertificateTransparencyVerificationRequired 返回 false
    }
    // 关键：Conscrypt 反射读取 TrustManager 的 getNetworkSecurityPolicy()，
    // 所以包装类必须暴露这个方法名
    class PolicyTrustManager(private val delegate: X509TrustManager) : X509TrustManager {
        private val policy = BgmPolicy()
        fun getNetworkSecurityPolicy(): NetworkSecurityPolicy = policy
        override fun checkClientTrusted(...) = delegate.checkClientTrusted(...)
        override fun checkServerTrusted(...) = delegate.checkServerTrusted(...)
        override fun getAcceptedIssuers(): Array<X509Certificate> = delegate.acceptedIssuers
    }
}
```

### 2.3 EchDoh.kt（DoH 解析 + 缓存 + 兜底）

```kotlin
internal object EchDoh {
    // 1) 节点池来自 BuildConfig（可配），缺省 Cloudflare Gateway（大陆可直连、返回 ech= 配置）
    // 2) 网关域名用内置 IP 直连：避免"DoH 解析自身"被污染的死循环
    private val dohClient = OkHttpClient.Builder()
        .dns(object : Dns {
            override fun lookup(hostname: String): List<InetAddress> =
                if (hostname == GATEWAY_HOST) GATEWAY_IPS   // 内置 IP，不查系统 DNS
                else runCatching { Dns.SYSTEM.lookup(hostname) }.getOrDefault(emptyList())
        }).build()

    // fetchA(host): 取 A 记录（type=A），缓存 60s，多节点轮询，全挂抛异常
    // fetchEch(host): 取 HTTPS 记录的 ech= 配置（type=HTTPS），缓存 30min
    //   解析: JSON 里 data 字段找 "ech=" 子串，取 base64 直到空格
    //   兜底: 目标站 CNAME 到 Cloudflare 时，DoH 全挂可用内置 ECH 配置（Base64 写死在代码里）
}
```

### 2.4 EchHttp.kt（双客户端 + Cookie 双向同步）

```kotlin
internal object EchHttp {
    fun get(): OkHttpClient  = build(followRedirects = true)   // 登录页/确认页链，自动跟随
    fun post(): OkHttpClient = build(followRedirects = false)  // 表单提交，手动读 Location（坑 #1）

    private fun build(followRedirects: Boolean): OkHttpClient {
        ConscryptEch.ensureProvider()
        val tm = ConscryptEch.PolicyTrustManager(ConscryptEch.systemTrustManager())
        val ctx = SSLContext.getInstance("TLSv1.3", "Conscrypt")
        ctx.init(null, arrayOf(tm), null)
        val factory = EchSocketFactory(ctx.socketFactory) { host -> EchDoh.fetchEch(host) }
        return OkHttpClient.Builder()
            .sslSocketFactory(factory, tm)
            .dns(EchDns)                                  // 目标站域名走 DoH 解析（坑 #4 用 hostname 精确匹配）
            .cookieJar(CookieManagerJar)                  // 与 WebView Cookie 双向同步
            .followRedirects(followRedirects)
            .followSslRedirects(followRedirects)
            .connectTimeout(15, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .build()
    }

    // 双向同步：ECH 请求的 Set-Cookie 写回 WebView CookieManager；
    // WebView 的 cookie 随 ECH 请求带上 → 验证码图片 GET 与表单 POST 会话一致（坑 #11）
    private object CookieManagerJar : CookieJar {
        override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>) {
            for (c in cookies) runCatching { CookieManager.getInstance().setCookie(url.toString(), c.toString()) }
            runCatching { CookieManager.getInstance().flush() }
        }
        override fun loadForRequest(url: HttpUrl): List<Cookie> {
            val raw = runCatching { CookieManager.getInstance().getCookie(url.toString()) }.getOrNull() ?: return emptyList()
            return raw.split(";").mapNotNull { s -> /* 解析 name=value → Cookie.Builder() */ }
        }
    }
}
```

### 2.5 EchLocalProxy.kt（进程内本地代理，核心）

```kotlin
object EchLocalProxy {
    const val PORT = 8888
    private const val TARGET_HOST = "<目标站点域名>"   // 如 bgm.tv

    private val pool = Executors.newCachedThreadPool { r -> Thread(r, "ech-local-proxy").apply { isDaemon = true } }
    private val cookieStore = LinkedHashMap<String, String>()  // 代理内 cookie（目标站域）

    fun start(): Boolean {  // ServerSocket(8888, 64, 127.0.0.1) + accept 线程，失败返回 false
        val ss = ServerSocket(PORT, 64, InetAddress.getByName("127.0.0.1"))  // 只监听本地，不对外
        running = true
        Thread({ acceptLoop(ss) }, "ech-local-proxy-accept").apply { isDaemon = true }.start()
        return true
    }

    // 每个请求（acceptLoop → pool.execute { handle(sock) }）：
    private fun handle(sock: Socket) {
        // 1. 读请求行 + headers + body（Content-Length）
        // 2. 组装 https://TARGET_HOST + rawPath，转发 headers：
        //    - host/content-length/cookie/connection/accept-encoding/transfer-encoding 不转发
        //    - referer / origin：重写 127.0.0.1 → https://TARGET_HOST（坑 #10，Discuz CSRF）
        //    - 其余原样转发
        // 3. 附加 Cookie 头（代理内 cookieStore）
        // 4. POST/PUT 带 body；否则 GET
        // 5. EchHttp.post() 执行（不跟随重定向）
        // 6. 响应：
        //    - Set-Cookie 全部存入 cookieStore
        //    - Location 重写：目标站内部 → 127.0.0.1；/相对路径 → 不动；外部回调域 → 原样放行
        //    - HTML：页面内 https://TARGET_HOST 绝对链接 → 127.0.0.1（相对路径天然命中，不动）
        //    - 重组响应头（重算 Content-Length），返回 HTTP/1.1 + Connection: close
    }
}
```

**关键点**：代理就是"网站"。WebView 打开 `http://127.0.0.1:8888/...` 后：
- 验证码图片 `/signup/captcha?...` → 相对路径自动命中代理 → ECH 转发 → 正常显示
- 表单 POST `/FollowTheRabbit` → 浏览器原生提交 → 代理收到**完整 body**（含 formhash/账号/密码/验证码/提交按钮）→ ECH 转发
- 页面 JS（含"提交时刷新验证码"）→ 浏览器原生执行，与服务器状态天然一致（坑 #8 消失）

### 2.6 EchInternalBrowserActivity.kt（WebView 登录页）

```kotlin
class EchInternalBrowserActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // 1. 取出 target URL，EchLocalProxy.start()，失败 finish()
        // 2. URL 重写: target.replace("https://<目标host>", "http://127.0.0.1:$PORT")
        // 3. WebView 配置:
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true
        CookieManager.getInstance().setAcceptCookie(true)
        CookieManager.getInstance().setAcceptThirdPartyCookies(webView, true)
        // 4. WebViewClient:
        override fun shouldOverrideUrlLoading(view, request): Boolean {
            val u = request.url?.toString() ?: return false
            if (u.startsWith("ani://")) {           // 自定义 scheme 回调（OAuth 完成）
                startActivity(Intent(Intent.ACTION_VIEW, request.url))  // 交给 App（App 注册了 intent-filter）
                finish()
                return true
            }
            return false   // 其余全部放行原生
        }
        // 5. loadUrl(localUrl)
    }
    override fun onDestroy() { EchLocalProxy.stop() }
}
```

### 2.7 接入点（AndroidManifest + 路由）

```xml
<!-- 登录页 Activity -->
<activity android:name=".EchInternalBrowserActivity" android:exported="false"
          android:theme="@android:style/Theme.NoTitleBar" />

<!-- 自定义 scheme 回调（OAuth 完成）：App 处理 code 的入口 Activity -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="ani" android:host="bangumi-oauth-callback" />
    </intent-filter>
</activity>
```

```kotlin
// 路由：App 的"打开浏览器"函数里拦截目标站
fun openBrowser(context: Context, url: String) {
    if (isTargetHost(url)) {   // hostname 精确匹配（坑 #4：不要 indexOf 整个 URL）
        context.startActivity(Intent(context, EchInternalBrowserActivity::class.java)
            .putExtra("url", url))
        return
    }
    // 其他 URL 走系统浏览器
}

// MainActivity 处理 ani:// 回调：把 code 提交给 App 后端完成绑定（App 轮询随后拿到 token）
private fun handleStartIntent(intent: Intent) {
    val data = intent.data ?: return
    if (data.host == "bangumi-oauth-callback") {
        val code = data.getQueryParameter("code") ?: return
        val state = data.getQueryParameter("state") ?: ""
        // 调后端提交接口，如 POST /v1/login/bangumi/oauth/callback { code, state }
    }
}
```

---

## 3. 登录形态适配对照

| 目标站登录形态 | 本方案行为 | 需要做什么 |
|---|---|---|
| Discuz 表单（bgm.tv） | 验证码/表单/JS 全原生 | 代理重写 Referer/Origin（坑 #10），其余零处理 |
| OAuth 授权页（已登录） | 页面原生显示"授权"按钮 | 零处理，点授权后 302 放行回调 |
| 自定义 scheme 回调（ani://） | WebView 不认 | shouldOverrideUrlLoading 拦截 → 交给 App → App 提交 code |
| 第三方跳转（付款/外部链接） | 302 到外部域 | 原样放行，WebView 原生跳转（或按需拦截） |

---

## 4. 踩坑清单（按时间线，AI 必读）

| # | 坑 | 现象 | 解法 |
|---|---|---|---|
| 1 | OkHttp 自动跟随 302 吞掉 Location | 登录成功但 App 拿不到跳转地址 | POST 用 `followRedirects(false)`，手动读 `Location` |
| 2 | 没有 CookieJar | 登录态丢失，页面反复跳回登录 | 统一 cookie 管理（本方案：CookieManager 双向同步） |
| 3 | `shouldInterceptRequest` 失败返回 null | WebView 明文直连被墙 → "连接被重置" | **fail-closed**：失败返回 502 错误页，绝不返回 null |
| 4 | 用 `indexOf(域名)` 匹配整个 URL | 页面里的统计参数（如 `dl=<目标站>...`）被误判成目标站请求，每次交互触发 DoH ECH → 卡顿 | **用 hostname 精确匹配**（Kotlin + JS 双重校验） |
| 5 | 登录 POST 缺 Referer 头 | 目标站防 CSRF 直接返回登录页 | 转发时带 `Referer: <登录页URL>` + `Origin: <目标站origin>` |
| 6 | 页面 `document.write` 覆盖注入的劫持脚本 | 劫持失效 | 避免依赖 document.write 之后的注入（本方案无注入，天然免疫） |
| 7 | FormData 不含提交按钮字段 | 服务器返回登录页（缺 `submit=登录` 之类的按钮字段） | 表单提交按钮 `<button name=value>` 不在 FormData 里，需补（本方案浏览器原生提交自带） |
| 8 | 目标站提交时刷新验证码 | JS 劫持提交：用户输入的验证码是旧的 → 服务器报"验证码错误" | 不要劫持！让浏览器原生提交（本方案） |
| 9 | 原生 Activity 拼表单 + GIF 验证码 | 图片区空白（GIF 解码不可靠）；动态字段拼不对 | 放弃原生表单，用 WebView 原生渲染 |
| 10 | **Referer/Origin 是 127.0.0.1** | 目标站报"来路不正确或验证字串不符"（CSRF 校验） | 代理转发时把 Referer/Origin 中的 `http://127.0.0.1:8888` **重写为真实域名的 https** |
| 11 | 验证码图片/session 一致性 | 图片 GET 与 POST 的 cookie 不一致 → 验证码错 | 图片 GET 和 POST 必须带**同一个 cookie**（本方案 CookieManager 双向同步） |
| 12 | Kotlin 编译：deprecation 当 error | `toHttpUrl` 未 import 编译失败 | 必须显式 `import okhttp3.HttpUrl.Companion.toHttpUrl` 后写 `url.toHttpUrl()` |
| 13 | WebView 打开自定义 scheme（ani://） | `ERR_UNKNOWN_URL_SCHEME` | shouldOverrideUrlLoading 拦截 ani:// → 交给 App（App 注册 intent-filter 处理） |
| 14 | ECH 策略不生效 | 请求直连（无 ECH）或全部报错 | Conscrypt Provider 必须插到 Security 第一位；TrustManager 必须暴露 `getNetworkSecurityPolicy()`；策略只对目标 host REQUIRED |

---

## 5. 验证流程（按顺序确认，每步有明确信号）

1. **DoH 通**：日志出现 `DoH A OK <host> -> <ip>`、`DoH ECH OK <host>`
2. **页面通**：WebView 显示登录页（不再是"连接被重置"）
3. **验证码通**：验证码图片正常显示（GET 与 POST 同 cookie）
4. **表单通**：登录 POST 返回 302（而不是登录页/错误页）
5. **CSRF 通**：不再报"来路不正确"（Referer 重写生效）
6. **授权通**：授权确认后 302 到回调地址
7. **回接通**：`ani://` 被 App 拦截，code 提交成功，App 轮询拿到 token

**日志规范**：ECH 组件全流程打日志（DoH 结果、代理请求行、302、错误），方便逐点定位。
发布正式版时按 buildType 关闭（见 §7）。

---

## 6. 其他要点

- **DoH 节点池**：可配置多节点（逗号分隔），缺省 Cloudflare Gateway（大陆可直连、返回 ech= 配置）
- **ECH fallback**：目标站 CNAME 到 Cloudflare 时，DoH 全挂可用内置 ECH 配置兜底（Base64 写死）
- **OAuth 后端模式（推荐）**：App 从自己的后端申请授权链接 → WebView 完成授权 → 302 回 App 后端绑定 → App 轮询拿 token。**App 只碰自己的后端，目标被墙站点只走 ECH 代理**
- **编译丢 GitHub Actions**：被墙站点在国内无法直接抓包/诊断，抓取分析必须在境外 runner 上做

---

## 7. 发布注意事项

- **固定签名**：生成一个长期 keystore（keytool -validity 36500），base64 存入 CI Secrets，
  workflow 解码后配置 `signing_release_*` 属性。**不要每次构建生成新 keystore**（否则用户升级报签名不一致）
- **日志开关**：ECH 日志按 `BuildConfig.DEBUG` 控制，release 自动关闭（不写文件、不打 logcat）
- **统计/埋点**：保持默认开启（播放事件、崩溃上报），不要误关

---

## 8. Go 版（同方法，弃用原因）

- Go 版：`gomobile bind` 起**独立进程**的 ECH 代理，WebView 同样打开 `http://127.0.0.1:8888/login`
- **方法完全一样**（本地代理 + WebView 原生），只是代理实现是 Go 独立进程
- **弃用**：独立进程会被 Android 系统回收（OOM/后台清理）→ 代理崩溃 → 登录中断
- **Conscrypt 版优势**：进程内线程，随 App 生命周期，系统杀不掉；无 gomobile AAR；更快更轻

---

## 9. 快速自查清单（AI 改完代码后对照）

- [ ] 代理只监听 127.0.0.1，不对外
- [ ] Conscrypt Provider 插到 Security 第一位（坑 #14）
- [ ] 转发用 Conscrypt + ECH（OkHttp sslSocketFactory + 自定义 DNS）
- [ ] Referer/Origin 已重写为真实域名（坑 #10）
- [ ] cookie 统一（CookieManager 双向同步），验证码 GET 与 POST 同 cookie（坑 #11）
- [ ] POST 不跟随重定向，手动读 Location（坑 #1）
- [ ] 302 回调域原样放行；HTML 绝对链接替换为 127.0.0.1，相对路径不动
- [ ] 自定义 scheme（ani://）被拦截交给 App（坑 #13）
- [ ] 不做任何 JS 劫持/表单构造（坑 #7 #8）
- [ ] 失败 fail-closed，绝不回落明文（坑 #3）
- [ ] hostname 精确匹配，不 indexOf 整 URL（坑 #4）
- [ ] 固定签名 + 日志开关 + 统计保持（§7）
