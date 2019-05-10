---
title: WebViewJavascriptBridge 原理
date: 2019-05-09 09:15:37
tags:
---
# 官方demo 交互流程

### 1.webview 加载 ExampleApp.html 
```javascript
 // ExampleApp.html 中的js代码
function setupWebViewJavascriptBridge(callback) {
        if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
        if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
        window.WVJBCallbacks = [callback];
       //创建iframe 梦开始的地方
        var WVJBIframe = document.createElement('iframe');
        WVJBIframe.style.display = 'none';
        WVJBIframe.src = 'https://__bridge_loaded__';
        document.documentElement.appendChild(WVJBIframe);
        setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
    }
```

###  2.创建 一个 iframe 来触发 原生webView的代理方法,即:
```objc
// UIWebView
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {}

//WKWebView
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {}
```

#### 上述两个代理方法调用的核心方法如下:
```objc
 //判断是否是自己的声明的scheme
 if ([_base isWebViewJavascriptBridgeURL:url]) {
 	//是否是 'https://__bridge_loaded__' 这个类型的url
    //如果是 就去加载 WebViewJavascriptBridge_JS.m 中的js代码
        if ([_base isBridgeLoadedURL:url]) {
            [_base injectJavascriptFile];
        } else if ([_base isQueueMessageURL:url]) {
            NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
            [_base flushMessageQueue:messageQueueString];
        } else {
            [_base logUnkownMessage:url];
        }
        return NO;
   }
```
### 到此为止是为了把WebViewJavascriptBridge_JS.m 中的js代码注入Dom中
---

## Native call js 

``` objc
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    if (data) {
        message[@"data"] = data;
    }
    //如果有回调 生成一个callbackId存到message里,并把回调存到全局变量responseCallbacks
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
}
// oc调用网页js方法 是通过WebViewJavascriptBridge._handleMessageFromObjC('%@')
// 给js传了一段json 
- (void)_dispatchMessage:(WVJBMessage*)message {
    NSString *messageJSON = [self _serializeMessage:message pretty:NO];
    ...
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    //主线程调用
    [self _evaluateJavascript:javascriptCommand];
}
```

### 再看 _handleMessageFromObjC()的实现
``` javascript
//处理来自oc发过来的message(json)
function _doDispatchMessageFromObjC() {
	//解析oc传过来的json
			var message = JSON.parse(messageJSON);
			var messageHandler;
			var responseCallback;
            //js调用oc   oc有回调
			if (message.responseId) {
				responseCallback = responseCallbacks[message.responseId];
				if (!responseCallback) {
					return;
				}
				responseCallback(message.responseData);
				delete responseCallbacks[message.responseId];
			} else {
				//oc调用js  js有回调
				if (message.callbackId) {
					var callbackResponseId = message.callbackId;
					responseCallback = function(responseData) {
						_doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
					};
				}
				
				var handler = messageHandlers[message.handlerName];
				if (!handler) {
					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
				} else {
					//调用已在 messageHandlers中注册的方法, 回调给responseCallback->_doSend
					//_doSend 即js调用oc 
					handler(message.data, responseCallback);
				}
			}
		}
```
---
## js call oc
  调用流程: WebViewJavascriptBridge.callHandler() -> _doSend()
```javascript
    function callHandler(handlerName, data, responseCallback) {
		if (arguments.length == 2 && typeof data == 'function') {
			responseCallback = data;
			data = null;
		}
		_doSend({ handlerName:handlerName, data:data }, responseCallback);
	}
    // 把js调用oc的参数 封装成message对象 并把回调保存起来
	function _doSend(message, responseCallback) {
		if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
			responseCallbacks[callbackId] = responseCallback;
			message['callbackId'] = callbackId;
		}
		sendMessageQueue.push(message);
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	}
```
### 到这又开始触发到oc webview的代理回调里去了,即这块代码:
```objc
 if ([_base isWebViewJavascriptBridgeURL:url]) {
        if ([_base isBridgeLoadedURL:url]) {
            [_base injectJavascriptFile];
        } else if ([_base isQueueMessageURL:url]) {
        	//这次是走的这, 去调用js WebViewJavascriptBridge._fetchQueue();
        	// 取到刚刚push到sendMessageQueue 里的json
            NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
            //处理js传过来的参数 如果有 callbackId 就说明js调oc的时候 oc是有回调的
            [_base flushMessageQueue:messageQueueString];
        } else {
            [_base logUnkownMessage:url];
        }
        return NO;
   }
```
## 至此整个调用流程就走通了 O(∩_∩)O哈哈~   


