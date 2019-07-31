---
layout: post
title: iOS - JSBridge for WKWebView
date: 2019-01-04
Author: Xiaoye
tags: [WebView]
excerpt_separator: <!--more-->
toc: true
---

WebView 和 JS 通信一般有两种方式

1. 第一种就是 [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 实现的方式，是通过在 js 新建一个 iframe，iframe 指定一个自定义 scheme 的 url，将该 frame 添加到当前 dom 中。那么就可以分别通过 WKWebView 的

   ```swift
   func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void)
   ```

   以及 UIWebView 的 

   ```swift
   func webView(_ webView: UIWebView, shouldStartLoadWith request: URLRequest, navigationType: UIWebView.NavigationType) -> Bool
   ```

   拦截到该 iframe 的 url，根据 url 的 scheme 再做具体操作，这样就可以进行通信了，具体实现可以看  WebViewJavascriptBridge 的源码。这种方式的好处就是同时适用 UIWebView、WKWebView，项目中若是想要同时兼容这两个 WebView，适用这种方式比较方便。

2. 第二种是直接通过原生 API 进行通信，UIWebView 通过 JSContext，WKWebView 通过 WKScriptMessageHandler。本文只讨论 WKWebView 相关



### 1. WKWebView 调用 JS

WKWebView 调用 js 主要通过以下方法

```swift
/* @abstract Evaluates the given JavaScript string.
 @param javaScriptString The JavaScript string to evaluate.
 @param completionHandler A block to invoke when script evaluation completes or fails.
 @discussion The completionHandler is passed the result of the script evaluation or an error.
*/
open func evaluateJavaScript(_ javaScriptString: String, completionHandler: ((Any?, Error?) -> Void)? = nil)
```

如以下的调用就是将 Object 以及 String 当做参数传递给 js 中的 `window.bridge.receiveMessage()` 方法

```swift
wkWebview.evaluateJavaScript("window.bridge.receiveMessage({'key': 'value'});", completionHandler: nil)

wkWebview.evaluateJavaScript("window.bridge.receiveMessage('message');", completionHandler: nil)
```

### 2. JS 调用 WKWebView

JS 调用 WKWebView 主要通过 JS 使用 `window.webkit.messageHandlers.name.postMessage(messageBody)` 方法

上面的 `name` 是指什么呢，`name` 表示在 WKWebView 中添加过的 scriptMessageHandler 的 name，scriptMessageHandler 类似一个 delegate，处理 WKWebView 接收到的 JS 的 message （即上述方法中的 messageBody），每个 scriptMessageHandler 对应一个 name

比如以下代码

```swift
// swift
wkWebview.configuration.userContentController.add(self, name: "bridge")
```

通过上述代码添加了一个 name 为 `bridge` 的 scriptMessageHandler 后，我们可以通过以下方式像 WKWebView 发送 message

```js
// js
window.webkit.messageHandlers.bridge.postMessage({
    "key": "value"
});
```

WKWebView 怎么接受这些 message 呢

通过实现协议 WKScriptMessageHandler 的方法接受 message

```swift
extension JustBridge: WKScriptMessageHandler {
    
    public func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        // body: ["key": "value"]
        guard let body = message.body as? [String: Any] else { return }
   
}
```

### 3. JSBridge 的实现

知道了 WKWebView 和 JS 间互相通信的方式，哪么我们就可以以此为基础封装一个 JSBridge

[JustBridge](https://github.com/Xiaoye220/JSBridge) 是我完成的一个适用于 WKWebView 的 `JSBridge` 。[JustBridge](https://github.com/Xiaoye220/JSBridge)  或在初始化的时候通过以下代码，在 webView 的 `atDocumentStart` 的时候向其中注入我们桥接的 [JavaScript](https://github.com/Xiaoye220/JSBridge/blob/master/JSBridge/Assets/bridge.js) 代码，因此无需我们另外向前端代码添加桥接 JS 代码。

```swift
fileprivate func injectBridgeJS() {
    let script = WKUserScript(source: JustBridge.bridge_js, injectionTime: .atDocumentStart, forMainFrameOnly: true)
    self.webview.configuration.userContentController.addUserScript(script)
}
```

使用 [JustBridge](https://github.com/Xiaoye220/JSBridge)  唯一需要做的就是在两端通过 `register` 注册一个操作，在另一端通过 `call` 调用之前注册的 handler 即可。

1. import JustBridge 并且声明一个 JustBridge 对象

```swift
import JustBridge
```

...

```swift
var bridge: JustBridge!
```

2. 通过 WKWebView 初始化一个 JustBridge

```swift
bridge = JustBridge(with: wkWebView)
```

3. Swift 注册一个 handler，或者请求一个 JS 方法。

```swift
// ==========
//   swift
// ==========
bridge.register("swiftHandler") { (data, callback) in
    print("[js call swift] - data: \(data ?? "nil")\n")
	callback("[response from swift] - response data: I'm swift response data")
}

bridge.call("jsHandler", data: data, callback: { responseData in
    print(responseData ?? "have no response data")
}, errorCallback: { errorMessage in
    // errorMessage 可以为 "HandlerNotExistError" 或 "DataIsInvalidError" 
    // 分别对应 call 的 handler 不存在的情况以及传入的 data 不为 array, dictinary, string, number 的情况
    print(errorMessage)
})
```

4. JS 注册一个 handler，或者请求一个 Swift 方法。

```js
// ==========
// javascript
// ==========
window.bridge.register("jsHandler", function(data, callback) {
    console.log("[swift call js] - data: " + JSON.stringify(data));
    callback("[response from js] - response data: I'm js response data");
});

window.bridge.call("swiftHandler", "hello world from js", function(responseData) {
    console.log(responseData.toString())
}, function(errorMessage) {
    console.log("error: " + errorMessage)
});
```