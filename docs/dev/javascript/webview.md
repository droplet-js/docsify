# WebView

Android 对 JavaScript 的支持相对简单。而 iOS 对 JavaScript 的支持就不那么简单了。

iOS 8 及以上就提供 WKWebView，然而到 iOS 11 及以上 WKWebView 才提供 Cookie 写入功能。 为了普通 HTTP 请求和 Web 页面请求保持 Cookie 的一致性，iOS 11 以下使用 UIWebView，iOS 11 及以上使用 WKWebView。

##### Android WebView

```java
webkit.addJavascriptInterface(new FakeWebkitTestJSAccessor(), FakeWebkitTestJSAccessor.NAME_SPACE);
```

```java
FakeWebkitTestJSAccessor {

    public static final String NAME_SPACE = "test";

    @JavascriptInterface
    public void message(String jsonStr) {
        Log.e("TAG", jsonStr);
    }
}
```

##### iOS UIWebView

```objectivec
//
//  FakeWebkitTestJSAccessor.h
//

#import <Foundation/Foundation.h>
#import <JavaScriptCore/JavaScriptCore.h>

NS_ASSUME_NONNULL_BEGIN

static NSString * const FAKE_WEBKIT_NAME_SPACE_TEST = @"test";

@protocol FakeWebkitTestJSExport <JSExport>

- (void)message:(NSString *)jsonStr;

@end

@interface FakeWebkitTestJSAccessor : NSObject<FakeWebkitTestJSExport>

@end

NS_ASSUME_NONNULL_END
```

```objectivec
//
//  FakeWebkitTestJSAccessor.m
//

#import "FakeWebkitTestJSAccessor.h"

@implementation FakeWebkitTestJSAccessor

- (void)message:(NSString *)jsonStr {
    NSLog(@"%@", jsonStr);
}

@end
```

```objectivec
JSContext * jsContext = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
jsContext[FAKE_WEBKIT_NAME_SPACE_TEST] = [[FakeWebkitTestJSAccessor alloc] init];
```

###### iOS WKWebView

```objectivec
#pragma mark WKScriptMessageHandler

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    NSDictionary * bodyParams = (NSDictionary *) message.body;

    NSString * function = [bodyParams objectForKey:@"function"];
    SEL selector = NSSelectorFromString([NSString stringWithFormat:@"%@:", function]);

    NSDictionary * parameters = [bodyParams objectForKey:@"parameters"];
    NSError * error = nil;
    NSData * jsonData = [NSJSONSerialization dataWithJSONObject:parameters options:NSJSONWritingPrettyPrinted error:&error];
    NSString * jsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];

    if ([FAKE_WEBKIT_NAME_SPACE_TEST isEqualToString:message.name]) {
        if ([_testJSAccessor respondsToSelector:selector]) {
            [_testJSAccessor performSelector:selector withObject:jsonStr];
        }
    } else {

    }
}
```

```objectivec
WKUserScript * noneSelectScript = [[WKUserScript alloc] initWithSource:javascript injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:YES];
[config.userContentController addUserScript:noneSelectScript];
[config.userContentController addScriptMessageHandler:self name:FAKE_WEBKIT_NAME_SPACE_TEST];
```

##### Html + JavaScript

```html
<!DOCTYPE html>
<html>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <meta name="viewport" content="width=device-width,target-densitydpi=high-dpi,initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
        <head>

            <title></title>
            <script type="text/javascript">

            function callMobile(handlerInterface,handlerMethod,parameters){
                 //handlerInterface由iOS addScriptMessageHandler与andorid addJavascriptInterface 代码注入而来。
                var dic = {'handlerInterface':handlerInterface,'function':handlerMethod,'parameters': parameters};

                if (/(iPhone|iPad|iPod|iOS)/i.test(navigator.userAgent)){
                    var ver = (navigator.appVersion).match(/OS (\d+)_(\d+)_?(\d+)?/);
                    ver = parseInt(ver[1], 10);
                    if (ver >= 11) {
                        window.webkit.messageHandlers[handlerInterface].postMessage(dic);
                    } else {
                        window[handlerInterface][handlerMethod](JSON.stringify(parameters));
                    }
                }else{
                    window[handlerInterface][handlerMethod](JSON.stringify(parameters));
                }
            }
            </script>
        </head>
        <body>
            <br>
            <br>
            <br>
            <br>
            <br>
            <br>

            <input type="button" value="打个招呼" onclick="callMobile('test','message',{'message':'你好么'})" />
        </body>
</html>
```