---
title: 无需安装SSL证书抓取国服舞萌和中二成绩
data: 2024/08/18 12:00:00
updated: 2024/08/18 16:55:00
categories:
  - "技术"
  - "Kotlin"
tags: 
  - "Maimai"
  - "Chunithm"
  - "音游"
  - "查分"
cover: https://moegirl.uk/images/4/43/Maimai_DX_BUDDiES_%E5%9B%BE%E6%A0%87.png
---

<div align=center>
    <img src="https://moegirl.uk/images/4/43/Maimai_DX_BUDDiES_%E5%9B%BE%E6%A0%87.png" alt="headImg">
</div>

<div align="center">

# 无需SSL证书抓取 `舞萌&中二` 成绩

</div>

首先叠个甲  
> 本文使用的方法与途径所有解释权归属[华立科技](https://www.wahlap.com/)  
> 请勿将本文的提到的技术用于非法用途

---

## 技术研究起因
由于目前已有的上传成绩到[水鱼查分器](https://www.diving-fish.com/maimaidx/prober/)或者是[落雪查分器](https://maimai.lxns.net/)的技术都依赖`Http代理`技术  
对于桌面端来说这个并不麻烦, 但是对于移动端设备, 特别是**未连接WLAN**的设备来说, 设置`Http代理`极其麻烦  
于是就想着使用Kotlin编写一个能够本地开启 Http代理 / VPN 并且能更方便的获取到成绩
便有了下文

## 技术的研究
一开始的时候, 咱一般都是使用[bakapiano](https://github.com/bakapiano)的[maimaidx-prober-proxy-updater](https://github.com/bakapiano/maimaidx-prober-proxy-updater)  
这个工具有两个优点:
- 它不需要安装SSL证书
- 基于Nodejs, 理论支持全平台

但是如前言所说, 这个依旧需要用户自己设置Http代理, 所以咱需要自行实现该工具的代理方式  
于是咱就从这个工具下手, 开始研究如何不安装SSL证书查询获取成绩的

通过对该项目源码与跟原开发者的沟通, 咱了解了它的实现方式  
那么maimaidx-prober-proxy-updater的无需SSL查分实现方式如下

> 1. 访问`https://tgk-wcaime.wahlap.com/wc_auth/oauth/authorize/[maimai-dx/chunithm]`
> > 以maimai-dx为例子  
> > 正常来说会跳转到 `https://open.weixin.qq.com/connect/oauth2/authorize?appid=$appid&redirect_uri=https://tgk-wcaime.wahlap.com/wc_auth/oauth/callback/maimai-dx?r=$token&response_type=code&scope=snsapi_base&state=$state&connect_redirect=1#wechat_redirect`  
> > 重要的是里面的redirect_url, 因为它带有token并会在微信浏览器中重定向到这个URL
> 2. 将`redirect_url`内的`url scheme`更改为 `http`
> 3. 将更改后的`oauthUtl`放到微信内访问
> 4. 通过Http代理捕捉redirect后的URL
> > 这一步捕捉的URL的`URL scheme`是http, 华立是没有给http协议进行任何处理的
> 5. 将该URL的`scheme`更改为`https`, 访问该URL并且保存cookie
> 6. 访问 `https://maimai.wahlap.com/maimai-mobile/home/` 是否成功
> 7. 访问 `https://maimai.wahlap.com/maimai-mobile/record/musicGenre/search/?genre=99&diff=$diff` 并获取HTML数据, 并上传到diving-fish查分器进行处理

那么以上便是bakapiano的web查分器实现原理

## Kotlin实现
以水鱼查分器为例子

完整代码前往 -> [Github Repo](https://github.com/SkyDynamic/MaiproberKotlinExample)

将会用到的库：
- Ktor

#### 入口点
```kotlin Main.kt
fun main() = runBlocking {
    println("oauthURL: ${getAuthUrl("maimai-dx")}")
    println("请尽快复制到微信中打开")
    startProxy()
}
```

#### 获取AuthUrl
```kotlin Main.kt
val client = HttpClient(CIO) {
    install(ContentNegotiation) {
        json()
    }
    install(HttpCookies) {
        storage = AcceptAllCookiesStorage()
    }
}

suspend fun getAuthUrl(type: String) : String {
    val resp: HttpResponse = client.get("https://tgk-wcaime.wahlap.com/wc_auth/oauth/authorize/$type")
    val url = resp.request.url.toString().replace("redirect_uri=https", "redirect_uri=http")
    return url
}
```

#### Ktor Http服务器
```kotlin ProxyServer.kt
// 新建一个NettyApplicationServer, 端口为2560
    embeddedServer(Netty, host = "0.0.0.0", port = 2560) {
        // 添加拦截成功页面
        routing {
            get("/success") {
                call.respond(HttpStatusCode.OK, "查询完成，请返回查分器查看")
            }
        }

        // 添加拦截处理
        intercept(ApplicationCallPipeline.Call) {
            val requestUrl = call.request.uri
            try {
                val uri = URI(requestUrl)
                // 如果拦截到的请求URL为 http 协议
                if (uri.scheme.equals("http")) {
                    // 如果拦截到的是 tgk-wcaime.wahlap.com
                    if (uri.host.equals("tgk-wcaime.wahlap.com")) {
                        // 跳转到 success 界面
                        call.respondRedirect("http://127.0.0.1:2560/success")
                        // 处理请求
                        onAuthHook(uri)
                    }
                } else
                    call.respond(HttpStatusCode.BadRequest, "Invalid URL")
                return@intercept
            } catch (_: Exception) {
            }
        }
    }.start(wait = true)
```

#### 拦截处理
```kotlin InterceptHandler.kt
suspend fun onAuthHook(authUrl: URI) {
    val urlString = authUrl.toString()
    // 修改这里的账号密码成你自己的
    val username = ""
    val password = ""

    // 将拦截的authUrl scheme改为 https
    val target = urlString.replace("http", "https")
    // 验证水鱼查分器账号密码
    if (verifyProberAccount(username, password)) {
        // 判断游戏类型
        if (target.contains("maimai-dx")) {
            // Maimai处理
            uploadMaimaiProberData(username, password, target)
        } else if (target.contains("chunithm")) {
            // 自行实现
        }
    } else {
        println("Prober账号密码错误")
    }
}
``` 

#### 水鱼查分器相关
```kotlin DivingFishProberUtil.kt
@Serializable
data class LoginResponse(val errcode: Int? = null, val message: String)

private const val loginUrl = "https://www.diving-fish.com/api/maimaidxprober/login"

// 随机延迟
suspend fun delayRandomTime(diff: Int) {
    val duration = 1000L * (diff + 1) + 1000L * 5 * Random.nextDouble()
    withContext(Dispatchers.IO) {
        delay(duration.toLong())
    }
}

// 验证水鱼查分器账号
suspend fun verifyProberAccount(username: String, password: String) : Boolean {
    val resp: HttpResponse = client.post(loginUrl) {
        headers {
            append(HttpHeaders.ContentType, "application/json;charset=UTF-8")
            append(HttpHeaders.Referrer, "https://www.diving-fish.com/maimaidx/prober/")
            append(HttpHeaders.Origin, "https://www.diving-fish.com")
        }
        contentType(ContentType.Application.Json)
        setBody("""{"username":"$username","password":"$password"}""")
    }
    val body: LoginResponse = resp.body()
    return body.errcode == null
}

suspend fun uploadMaimaiProberData(
    username: String,
    password: String,
    authUrl: String
) {
    println("开始更新Maimai成绩")

    // 登录MaimaiDX主页并保存cookie
    println("登录MaimaiDX主页...")
    client.get(authUrl) {
        headers {
            append(HttpHeaders.Connection, "keep-alive")
            append("Upgrade-Insecure-Requests", "1")
            append(HttpHeaders.UserAgent, "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) " +
                    "Chrome/81.0.4044.138 Safari/537.36 NetType/WIFI " +
                    "MicroMessenger/7.0.20.1781(0x6700143B) WindowsWechat(0x6307001e)")
            append(HttpHeaders.Accept, "text/html,application/xhtml+xml,application/xml;q=0.9," +
                    "image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9")
            append("Sec-Fetch-Site", "none")
            append("Sec-Fetch-Mode", "navigate")
            append("Sec-Fetch-User", "?1")
            append("Sec-Fetch-Dest", "document")
            append(HttpHeaders.AcceptEncoding, "gzip, deflate, br")
            append(HttpHeaders.AcceptLanguage, "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7")
        }
    }

    val result = client.get("https://maimai.wahlap.com/maimai-mobile/home/")

    if (result.bodyAsText().contains("错误")) {
        throw RuntimeException("登录公众号失败")
    }

    // 难度列表
    val diffNameList = listOf(
        "Basic",     // diff = 0
        "Advanced",  // diff = 1
        "Expert",    // diff = 2
        "Master",    // diff = 3
        "Re:Master"  // diff = 4
    )

    var diff = 0
    for (diffName in diffNameList) {
        println("获取 Maimai-DX $diffName 难度成绩数据")
        delayRandomTime(diff)

        with(client) {
            // 获取成绩HTML数据
            val scoreResp: HttpResponse = get(
                "https://maimai.wahlap.com/maimai-mobile/record/musicGenre/search/?genre=99&diff=$diff"
            )
            val body = scoreResp.bodyAsText()

            val data = Regex("<html.*>([\\s\\S]*)</html>")
                .find(body)?.groupValues?.get(1)?.replace("\\s+/g", " ")

            println("上传 Maimai-DX $diffName 难度成绩到 Diving-Fish 查分器数据库")

            // 上传HTML数据到Diving-Fish
            val resp: HttpResponse = post("https://www.diving-fish.com/api/pageparser/page") {
                headers {
                    append(HttpHeaders.ContentType, "text/plain")
                }
                contentType(ContentType.Text.Plain)
                setBody("""<login><u>$username</u><p>$password</p></login>$data""")
            }
            val respData: String = resp.bodyAsText()

            println("Diving-Fish 上传 Maimai-DX $diffName 分数接口返回信息: $respData")
        }
        diff += 1
    }
    println("Maimai 成绩上传到 Diving-Fish 查分器数据库完毕")
}
```

## 特别感谢
[@bakapiano](https://github.com/bakapiano) - 提供了很棒的查分思路  
[@ZhuRuoLing](https://github.com/zHUrUOLING) - 帮助咱编写了部分逻辑

[@Jetbrains](https://www.jetbrains.com/) - 制作了非常棒的`Intellij Idea` 让咱编写代码十分方便