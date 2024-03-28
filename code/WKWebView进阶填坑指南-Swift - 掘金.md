# WKWebView进阶填坑指南-Swift - 掘金
Cookie处理
--------

在设置`Cookie`的时候，我们经常做的是在请求的请求头里添加 `Cookie`，但是这只是把`Cookie`发送给了服务端，我们本地并没有保存 `Cookie`，`Cookie` 最终要写到`WebView`的一个`Cookie`文件目录里面，后续`WebView`里面自己的发起的请求或者跳转才能在发起请求的时候在对应的域名下面取到`Cookie`传出去。

`Webview` 加载 H5 页面，实际上是把页面相关的`.html、js、css`文件下载到本地，然后再加载，这时页面去获取 `Cookie` 的时候，是去本地`WebView`里的`Cookie`文件目录里查找，如果没有设置的话肯定就找不到了。所以在设置`Cookie`的时候，服务端和客户端都要设置。

#### 一、服务端 Cookie 设置

在使用`UIWebView`的时候，我们是通过 `NSHTTPCookieStorage`来管理 `Cookie` 的，下面我们给`devqiaoyu.tech`这个域名添加一个名为`user`的`Cookie`。

```ini
var props = Dictionary<HTTPCookiePropertyKey, Any>()
props[HTTPCookiePropertyKey.name] = "user"
props[HTTPCookiePropertyKey.value] = "admin"
props[HTTPCookiePropertyKey.path] = "/"
props[HTTPCookiePropertyKey.domain] = "devqiaoyu.tech"
props[HTTPCookiePropertyKey.version] = "0"
props[HTTPCookiePropertyKey.originURL] = "devqiaoyu.tech"
if let cookie = HTTPCookie(properties: props) {
    HTTPCookieStorage.shared.setCookie(cookie)
}

```

**`WKWebView Cookie`问题在于`WKWebView`发起的请求不会自动带上存储于`NSHTTPCookieStorage`容器中的`Cookie`。** 

解决办法也很简单，就是在`WKWebView`发起请求之前，先从`NSHTTPCookieStorage`读取`Cookie`，然后手动往`URLRequest`的请求头里添加一下`Cookie`。

```swift
func getCookie() -> String {
    var cookieString = ""
	if let cookies = HTTPCookieStorage.shared.cookies {
		for cookie in cookies {
		if cookie.domain == cookieDomain {
			let str = "\(cookie.name)=\(cookie.value)"
			cookieString.append("\(str);")
		}
	}
	return cookieString
}

var request = URLRequest(url: URL(string: "https://devqiaoyu.com"))
request.addValue(getCookie(), forHTTPHeaderField: "Cookie")

```

当服务器页面发生重定向的时候，此时第一次在`RequestHeader`中写入的`Cookie`会丢失，还需要对重定向的请求重新做添加`Cookie`的处理。具体方法请往下看~

#### 二、客户端 Cookie 设置

上面这么写完了，当页面加载的时候，后端无论是啥语言，都能从请求头里看到`Cookie`了，但是后端渲染返回页面后，在客户端的`WebView`里运行的时候，JS 在执行的时候调用 `document.cookie` API 是读取不到`Cookie`的，所以还得针对客户端`Cookie`进行处理。

```ini
var cookieString = ""
if let cookies = HTTPCookieStorage.shared.cookies {
	for cookie in cookies {
		if cookie.domain == "devqiaoyu.tech" {
			let str = "\(cookie.name)=\(cookie.value)"
			cookieString.append("document.cookie='\(str);path=/;domain=devqiaoyu.tech'
		}
	}
}
let cookieScript = WKUserScript(source: cookieString, injectionTime: .atDocumentStart, forMainFrameOnly: false)
let userContentController = WKUserContentController()
userContentController.addUserScript(cookieScript)

let webViewConfig = WKWebViewConfiguration()
webViewConfig.userContentController = userContentController

let webV = WKWebView(frame: CGRect.zero, configuration: webViewConfig)

```

**客户端`Cookie`注入实际上就是创建一个 JS 脚本，让`WebView`去执行，推荐在`.atDocumentStart`这个时机进行预置静态 JS 的注入。**这样`WebView`在加载后端返回的静态页面的时候，就可以拿到保存着客户端的`Cookie`了。

> **注意：document.cookie() 无法跨域设置 Cookie**，比如你第一次加载的请求时 [www.baidu.com](https://link.juejin.cn/?target=http%3A%2F%2Fwww.baidu.com "http://www.baidu.com") ，在重定向的时候跳转到了 [www.google.com](https://link.juejin.cn/?target=http%3A%2F%2Fwww.google.com "http://www.google.com") ，那么第二个请求就可能因为没有携带 `Cookie`而无法访问。当然啦，解决办法还是有的，请往下看~

URL拦截
-----

在`WKWebView`中，每一次页面跳转之前，都会调用下面的回调函数：

```less
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void)

```

#### Web 页面重定向问题

重定向问题有两种：

*   服务器页面重定向，需要对新发起的请求重新种`Cookie`
*   本地页面重定向，只要客户端设置了`Cookie`，那么就不需要处理了

所以如果是服务器页面重定向，那么判断此时`Request`是否有你要的`Cookie`没有就`Cancel`掉，修改`Request` 重新发起。

```less
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void)
{
    var shouldCancelLoadURL = false
    if let cookie = navigationAction.request.value(forHTTPHeaderField: "Cookie") {
        if cookie.contains("user") {
            shouldCancelLoadURL = false
        } else {
            var request = URLRequest(url: URL(string: (navigationAction.request.url?.absoluteString)!)!)
            request.addValue(getCookie(), forHTTPHeaderField: "Cookie")
            webView.load(request)
            shouldCancelLoadURL = true
        }
    } else {
        var request = URLRequest(url: URL(string: (navigationAction.request.url?.absoluteString)!)!)
        request.addValue(getCookie(), forHTTPHeaderField: "Cookie")
		webView.load(request)
		shouldCancelLoadURL = true
    }
    
    if shouldCancelLoadURL {
    	decisionHandler(WKNavigationActionPolicy.cancel)
	} else {
		decisionHandler(WKNavigationActionPolicy.allow)
	}
}

```

#### 跨域问题

针对跨域的问题，解决办法和上面的方法类似，仅仅是判断条件不同。

```swift
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void)
{
    var shouldCancelLoadURL = false
    if let url = navigationAction.request.url?.absoluteString {
        if url.contains("devqiaoyu.tech") { 
            shouldCancelLoadURL = false
        } else {
        	
            shouldCancelLoadURL = true
        }
    } else {
    	
		shouldCancelLoadURL = true
    }
    
    if shouldCancelLoadURL {
    	decisionHandler(WKNavigationActionPolicy.cancel)
	} else {
		decisionHandler(WKNavigationActionPolicy.allow)
	}
}

```

#### 假跳转的请求拦截

**一种 JS 调用 Native 的通信方案**，详细介绍可以看[从零收拾一个hybrid框架（一）-- 从选择JS通信方案开始](https://link.juejin.cn/?target=http%3A%2F%2Fwww.cocoachina.com%2Fios%2F20180109%2F21795.html "http://www.cocoachina.com/ios/20180109/21795.html")。下面内容是从该文章内摘录的。

何谓 **假跳转的请求拦截** 就是由网页发出一条新的跳转请求，跳转的目的地是一个非法的压根就不存在的地址，比如

```arduino

https:

wakaka:

```

看我下面写的那条假跳转地址，这么一条什么都不是的扯淡地址，直接放到浏览器里，直接扔到`WebView`里，肯定是妥妥的什么都打不开的，而如果在经过我们改造过的`Hybrid WebView`里，进行拦截不进行跳转

url 地址分为这么几个部分

*   协议：也就是 `http/https/file` 等，上面用了 `wakaka`
*   域名：上面的 `wenku.baidu.com` 或 `wahahalalala`
*   路径：上面的 `xxxx?` 或 `action？`
*   参数：上面的 `xx=xx` 或 `param=paramobj`

如果我们构建一条假url

*   用协议与域名当做通信识别
*   用路径当做指令识别
*   用参数当做数据传递

客户端会无差别拦截所有请求，真正的 url 地址应该照常放过，只有协议域名匹配的 url 地址才应该被客户端拦截，拦截下来的 url 不会导致 WebView 继续跳转错误地址，因此无感知，相反拦截下来的 url 我们可以读取其中路径当做指令，读取其中参数当做数据，从而根据约定调用对应的 Native 原生代码

以上其实是一种 **协议约定** 只要 JS 侧按着这个约定协议生成假 url，Native 按着约定协议拦截/读取假 url，整个流程就能跑通。

User-Agent设置
------------

#### 全局设置

就是`App`内所有`Web`请求的`User-Agent`全部被修改。

```ini
// UIWebView
let webView = UIWebView(frame: CGRect.zero)
let userAgent = webView.stringByEvaluatingJavaScript(from: "navigator.userAgent")
if let agent = userAgent {
    let user = "@\(agent);extra_user_agent"
    let dict = ["UserAgent":user]
    UserDefaults.standard.register(defaults: dict)
}

// WKWebView
let webV = WKWebView(frame: CGRect.zero)
webV.evaluateJavaScript("navigator.userAgent") { (result, error) in
	if let oldAgent = result as? String {
		let user = "@\(oldAgent);extra_user_agent"
		let dict = ["UserAgent":user]
		UserDefaults.standard.register(defaults: dict)
	}
}

```

### 单个WebView设置

在`iOS9`，`WKWebView`提供了一个非常便捷的属性去更改`User-Agent`，就是`customUserAgent`属性。这样使用起来不仅方便，也不会全局更改`User-Agent`，可惜的是`iOS9`才有，如果适配`iOS8`，还是要使用上面的方法。

```ini
let webView = UIWebView(frame: CGRect.zero)
let userAgent = webView.stringByEvaluatingJavaScript(from: "navigator.userAgent")
if let agent = userAgent {
    let user = "@\(agent);extra_user_agent"
    webView.customUserAgent = user
}

```

参考文章
----

[WKWebView 那些坑](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F06963b57d78e "https://www.jianshu.com/p/06963b57d78e")