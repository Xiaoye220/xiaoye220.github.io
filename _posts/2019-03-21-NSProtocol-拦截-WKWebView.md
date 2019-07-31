---
layout: post
title: iOS - NSProtocol 拦截 WKWebView  POST 请求 body 会被清空的问题解决
date: 2019-03-21
Author: Xiaoye
tags: [WebView, iOS]
excerpt_separator: <!--more-->
toc: true
---

> 由于 WKWebView 在独立进程里执行网络请求。一旦注册 http(s) scheme 后，网络请求将从 Network Process 发送到 App Process，这样 NSURLProtocol 才能拦截网络请求。在 webkit2 的设计里使用 MessageQueue 进行进程之间的通信，Network Process 会将请求 encode 成一个 Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode 的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了 —— 摘自 [WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)

针对以上问题我使用过以下两种方案，并且测试过都是可行的

#### （1）方案一

我们通过 URLProtocol 拦截的 WKWebView 的时候可以不拦截 http、https，而是拦截一个指定的 customScheme。并且将 html 中所有 GET 请求的 url 的 scheme 都替换成 customScheme，POST 请求还是保持 http、https 的 scheme。下面是一个实现流程，很多代码略过了，只是说明方案。

1. 下载所有的 html、css、js、图片 至沙盒

2. NSURLProtocol registerScheme

```objc
[NSURLProtocol wk_registerScheme:@"customScheme"];
```

3. 打开页面 url，假设我们打开的 url 为 `https://www.baidu.com`，通过以下代码我们就打开了一个 `customScheme://www.baidu.com` 了，这个请求就可以被 NSURLProtocol 拦截

```objc
NSString *url = @"https://www.baidu.com";
[url stringByReplacingOccurrencesOfString:@"https" withString:@"customScheme"];

NSURLRequest *request = [[NSURLRequest alloc] initWithURL:[NSURL URLWithString:url]];
WKWebView *webview = [[WKWebView alloc] init];
[webview loadRequest:request];
```

4. 在 NSURLProtocol 中判断是否是 `customScheme` 并且是我们首页 `index.html`  的地址，是的话结果该请求，获取本地的 html 数据，手动将内部的 html、css、js、图片 的 url 替换成  `customScheme` 

```objc
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    NSString *urlStr = request.URL.absoluteString;
    if([urlStr isEqualToString:@"customScheme://www.baidu.com"]) {
        return YES；
    }
    return NO；
}

- (void)startLoading {
    NSString *urlStr = request.URL.absoluteString;
    if([urlStr isEqualToString:@"customScheme://www.baidu.com"]) {
        return YES；
    }
    
    // 从本地获取缓存的 https://www.baidu.com 文件的内容
    NSString *indexHtml = ...;
    // 将 indexHtml 中 css、js、图片 的 url 的 https 替换成 customScheme
    NSString *customSchemeHtml = [indexHtml convertToCustomSchemeHtml];
    
	NSData *htmlData = [customSchemeHtml dataUsingEncoding:NSUTF8StringEncoding];
    NSURLResponse *response = ...;
    
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    [self.client URLProtocol:self didLoadData:htmlData];
    [self.client URLProtocolDidFinishLoading:self];
}
```

5. 那么 `https://www.baidu.com` 中的静态资源的 scheme 都替换后就可以通过 NSURLProtocol 拦截到了，拦截到后再从本地取缓存返回 WKWebView。同时因为没有拦截 http、https ，所有的 post 请求都不会被 NSURLProtocol 拦截，还是由 WKWebView 自己处理，那么就不会有 Body 丢失的问题

#### （2）方案二

这种方案又有两种实现
1. 一种是前端手动将 POST 请求改为 JS 调用 Native 的方式，这样的话就将工作量转交给前端开发的朋友的。同时对于移动端和 PC 端都支持的 H5 页面需要前端做适配，避免在 PC 端进行 POST 请求也是用  JS 调用 Native 的方式，请求不到数据。

2. 另一种就是客户端注入一段 HookAjax 的 JS 代码，拦截所有的 XMLHttpRequest 的 POST 请求转移给客户端处理。HookAjax 后也有两种方案处理该 POST 请求
    * 将 POST 请求的 body 装在 header 中，NSURLProtocol 拦截到的 POST 请求可以从 header 中获取到实际的 body 数据。但是这样子有个问题，header 的大小是有限制的，会有局限性。但是实现会方便很多。
    * 将 POST 请求通过 JS 和 Native 交互的方式将请求转交给 Native 处理并且在 Native 处理完后将结果返回给 JS，我下面介绍的就是这种方式的实现

下面是我的 HookAjax 的实现，主要参考了 [Ajax-hook](https://github.com/wendux/Ajax-hook) 并且做了一定的修改以更好的处理我们的情况。

```js
function hookAjax(proxy) {
    // 保存真正的XMLHttpRequest对象
    window._ahrealxhr = window._ahrealxhr || XMLHttpRequest;
    XMLHttpRequest = function() {
        var xhr = new window._ahrealxhr;
        // 直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象
        Object.defineProperty(this, 'xhr', {
            value: xhr
        })
    };
    
    // 获取 XMLHttpRequest 对象的属性
    var prototype = window._ahrealxhr.prototype;
    for (var attr in prototype) {
        var type = "";
        try {
            type = typeof prototype[attr]
        } catch (e) {}
        if (type === "function") {
            XMLHttpRequest.prototype[attr] = hookfunc(attr);
        } else {
            // 给属性提供 getter、setter 方法
            Object.defineProperty(XMLHttpRequest.prototype, attr, {
                get: getFactory(attr),
                set: setFactory(attr),
                enumerable: true
            })
        }
    }
    
    function getFactory(attr) {
        return function() {
            // 判断对象是否包含特定的自身（非继承）属性
            var v = this.hasOwnProperty(attr + "_") ? this[attr + "_"] : this.xhr[attr];
            var attrGetterHook = (proxy[attr] || {})["getter"];
            return attrGetterHook && attrGetterHook(v, this) || v
        }
    }
    
    function setFactory(attr) {
        return function(v) {
            var xhr = this.xhr;
            var that = this;
            var hook = proxy[attr];
            if (typeof hook === "function") {  // 回调属性 onreadystatechange 等
                xhr[attr] = function() {
                	// ========================  修改 1 ==================
                    hook.call(that, xhr) || v.apply(xhr, arguments);
                }
            } else {
                //If the attribute isn't writeable, generate proxy attribute
                var attrSetterHook = (hook || {})["setter"];
                v = attrSetterHook && attrSetterHook(v, that) || v;
                
                // ========================  修改 2 ==================
                xhr[attr] = v;
                this[attr + "_"] = v;
            }
        }
    }
    
    function hookfunc(func) {
        return function() {
            var args = [].slice.call(arguments);
            
            // call() 方法调用一个函数, 其具有一个指定的this值和分别地提供的参数
            // 该方法的作用和 apply() 方法类似，只有一个区别，就是call()方法接受的是若干个参数的列表，而apply()方法接受的是一个包含多个参数的数组
            if (proxy[func]) {
                // ========================== 修改 3 ===============
                var result = proxy[func].call(this, args, this.xhr);
                if(result) {
                    return result;
                }
            }
            
            return this.xhr[func].apply(this.xhr, args);
        }
    }
    
    return window._ahrealxhr;
}
```

针对的修改主要有上面注释的 3 个修改

* `修改 1` : 改变 hook 的 `onreadystatechange`、`onload` 这类方法的参数，统一参数
* `修改 2` : 强制通过 `_ + 属性名` 的方式添加属性到我们修改后的  `XMLHttpRequest` 的对象中，因为有一些属性是只读属性，用  [Ajax-hook](https://github.com/wendux/Ajax-hook)  的方式我们没办法设置这些属性
* `修改 3` : 对于 `getAllResponseHeaders` 、`getResponseHeader` 返回指定结果



接下来还要添加我们拦截 Ajax Post 请求到 Native 执行的 JS 代码

```js
window.MyAjax = {
    hookedXHR: {},
    hookAjax: hookAjax,
    nativePost: nativePost,
    nativeCallback: nativeCallback
};
        
// 添加请求 Native 的代码
function nativePost(xhrId, params) {
    // TODO: 请求 Native
}

// Native 请求完成后调用该 JS 方法并且传入指定参数
function nativeCallback(xhrId, statusCode, responseText, responseHeaders, error) {
    var xhr = window.MyAjax.hookedXHR[xhrId];
    
    if(xhr.isAborted) { // 如果该请求已经手动取消了
        return;
    }
    
    if(error) {
        xhr.readyState = 1;
        if(xhr.onerror) {
            xhr.onerror();
        }
    } else {
        xhr.status = statusCode;
        xhr.responseText = responseText;
        xhr.readyState = 4;
        
        xhr.myResponseHeaders = responseHeaders;
        
        if(xhr.onreadystatechange) {
            xhr.onreadystatechange();
        }
        if(xhr.onload) {
            xhr.onload();
        }
    }
}

// hook ajax 方法
window.MyAjax.hookAjax({
    // 设置 RequestHeader 将参数保存，在 send 中将其一起发送给 Native
    setRequestHeader: function (arg, xhr) {
        if(!this.myHeaders) {
            this.myHeaders = {};
        }
        this.myHeaders[arg[0]] = arg[1];
    },
    getAllResponseHeaders: function (arg, xhr) {
        var headers = this.myResponseHeaders;
        if(headers) {
            if(typeof(headers) === 'object') {
                var result = '';
                for(var key in headers) {
                    result = result + key + ':' + headers[key] + '\r\n'
                }
                return result;
            }
            return headers;
        }
    },
    getResponseHeader: function (arg, xhr) {
        if(this.myResponseHeaders && this.myResponseHeaders(arg[0])) {
            return this.myResponseHeaders(arg[0]);
        }
    },
    // 保存 open 中的参数，在 send 中判断是否为 POST 方法
    open: function (arg, xhr) {
        this.myOpenArg = arg;
    },
    send: function (arg, xhr) {
        this.isAborted = false;
        if(this.myOpenArg[0] === 'POST') {
            var params = {};
            params.data = arg[0];
            params.method = 'POST';
            params.header = this.myHeaders;

            var url = this.myOpenArg[1];
            var location = window.location;
            if(!url.startsWith(location.protocol)) {
                url = location.origin + url;
            }
            params.url = url;

			// XMLHttpRequest 的标识符，用于 Native 返回时确认时哪个 XMLHttpRequest
            var xhrId = 'xhrId' + (new Date()).getTime();
            window.MyAjax.hookedXHR[xhrId] = this;
            
            window.MyAjax.nativePost(xhrId, params);

            // 通过 return true 可以阻止默认 Ajax 请求，不返回则会继续原来的请求
            return true;
        }
    },
    abort: function (arg, xhr) {
        if(this.myOpenArg[0] === 'POST') {
            if(xhr.onabort) {
                xhr.onabort()
            }
            return true;
        }
    }
    
});
```

上面的代码和 Native 进行交互的代码就是 `nativePost` 和 `nativeCallback` 两个方法，一个需要将 `xhrId` 和该 POST 请求相关的数据传给 Native，一个需要 Native 执行完请求后将  `xhrId`  和结果放回给 JS。这样子一个完整的流程就走通了。

这里贴一个例子[Ajax-hook iOS Demo](https://github.com/Xiaoye220/JavaScript/tree/master/Ajax-hook)，用的是我自己的 [JSBridge](https://github.com/Xiaoye220/JSBridge) 作为 JS、Native 的桥接库实现的一个 Demo

将上述两端 JS 代码注入 WebView，就可以实现一个 Post 请求的拦截了。

**题外话：**通过原生 NSURLSession 请求时，如果用的是 `NSURLSession.sharedSession` ，那么该请求是会被 NSURLProtocol 拦截到的，打断点后发现 NSURLProtocol 拦截到的 POST 请求的 bady 也是 nil，因为 body 数据这时候已经被转成 stream 了，可以在 `request.HTTPBodyStream` 中解析它

***



> 参考资料
>
> [Ajax-hook](https://github.com/wendux/Ajax-hook) 