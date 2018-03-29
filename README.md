This is what I learned about `WKWebView`, Apple's new WebKit API debuted on iOS 8.

As of this writing, the latest iOS version is iOS 8.1.3.

## `file:///` doesn't work without `tmp` directory
Only the `tmp` directory access can be accessed with the `file:` scheme, as of iOS 8.0.2.

You can see what directory access is allowed on [the shazron / WKWebViewFIleUrlTest GitHut repo](https://github.com/shazron/WKWebViewFIleUrlTest).

Note: On iOS 9.0.0 (beta), you can use the below method to load files from Documents, Library and tmp folders. But App Bundle cannot.
`- (nullable WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL NS_AVAILABLE(10_11, 9_0);`

## Can't handle in Storyboard and Interface Builder
You need to set `WKWebView` and any `NSLayoutConstraint`s programmatically.

## HTML `<a>` tag with `target="_blank"` won't respond
See Stack Overflow: [Why is WKWebView not opening links with target=“_blank”](http://stackoverflow.com/questions/25713069/why-is-wkwebview-not-opening-links-with-target-blank)


## URL Scheme and App Store links won't work

#### Example

```objc
// Using [bendytree/Objective-C-RegEx-Categories](https://github.com/bendytree/Objective-C-RegEx-Categories) to check URL String
#import "RegExCategories.h"

- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler
{
    NSURL *url = navigationAction.request.URL;
    NSString *urlString = (url) ? url.absoluteString : @"";

    // iTunes: App Store link
    if ([urlString isMatch:RX(@"\\/\\/itunes\\.apple\\.com\\/")]) {
        [[UIApplication sharedApplication] openURL:url];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    // Protocol/URL-Scheme without http(s)
    else if (![urlString isMatch:[@"^https?:\\/\\/." toRxIgnoreCase:YES]]) {
        [[UIApplication sharedApplication] openURL:url];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    decisionHandler(WKNavigationActionPolicyAllow);
}
```


## alert, confirm, prompt from JavaScript needs to set `WKUIDelegate` methods

If you want to show dialog boxes, you have to implement the following methods:

`webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:`
`webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:`
`webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:`

[Here is how to set those](http://qiita.com/ShingoFukuyama/items/5d97e6c62e3813d1ae98). 


## Basic/Digest/etc authentication input dialog boxes need to set a `WKNavigationDelegate` method

If you want to present an authentication challenge to user, you have to implement the method below:

`webView:didReceiveAuthenticationChallenge:completionHandler:`

#### Example:

```objc
- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler
{
    NSString *hostName = webView.URL.host;
    
    NSString *authenticationMethod = [[challenge protectionSpace] authenticationMethod];
    if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodDefault]
        || [authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPBasic]
        || [authenticationMethod isEqualToString:NSURLAuthenticationMethodHTTPDigest]) {
        
        NSString *title = @"Authentication Challenge";
        NSString *message = [NSString stringWithFormat:@"%@ requires user name and password", hostName];
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:title message:message preferredStyle:UIAlertControllerStyleAlert];
        [alertController addTextFieldWithConfigurationHandler:^(UITextField *textField) {
            textField.placeholder = @"User";
            //textField.secureTextEntry = YES;
        }];
        [alertController addTextFieldWithConfigurationHandler:^(UITextField *textField) {
            textField.placeholder = @"Password";
            textField.secureTextEntry = YES;
        }];
        [alertController addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction *action) {
            
            NSString *userName = ((UITextField *)alertController.textFields[0]).text;
            NSString *password = ((UITextField *)alertController.textFields[1]).text;
            
            NSURLCredential *credential = [[NSURLCredential alloc] initWithUser:userName password:password persistence:NSURLCredentialPersistenceNone];
            
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
            
        }]];
        [alertController addAction:[UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) {
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }]];
        dispatch_async(dispatch_get_main_queue(), ^{
            [self presentViewController:alertController animated:YES completion:^{}];
        });
        
    }
    else if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // needs this handling on iOS 9
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
        // or, see also http://qiita.com/niwatako/items/9ae602cb173625b4530a#%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%B3%E3%83%BC%E3%83%89
    }
    else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
}
```


## Cookie sharing between multiple `WKWebView`s

Use a `WKProcessPool` to share cookies between web views.

#### Example:

```objc
self.processPool = [[WKProcessPool alloc] init];

WKWebViewConfiguration *configuration1 = [[WKWebViewConfiguration alloc] init];
configuration1.processPool = self.processPool;
WKWebView *webView1 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration1];
...
WKWebViewConfiguration *configuration2 = [[WKWebViewConfiguration alloc] init];
configuration2.processPool = self.processPool;
WKWebView *webView2 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration2];
...
```

See [this Stack Overflow question](http://stackoverflow.com/questions/25797972/cookie-sharing-between-multiple-wkwebviews).


## Cannot work with `NSURLProtocol`, `NSCachedURLResponse`, `NSURLCache`
`UIWebView` can filter ad URLs and cache to read websites offline using `NSURLProtocol`, `NSURLCache`, and `NSCachedURLResponse`.

But `WKWebView` cannot work with those APIs.


## Cookie, Cache, Credential, WebKit data cannot easily delete

### iOS 8
After much trial and error, I've reached the following conclusion:

1. Use `NSURLCache` and `NSHTTPCookie` to delete cookies and caches in the same way as you used to do on `UIWebView`.
2. If you use `WKProccessPool`, re-initialize it.
3. Delete `Cookies`, `Caches`, `WebKit` subdirectories in the `Library` directory.
4. Delete all `WKWebView`s

### iOS 9

```objc
//// Optional data
NSSet *websiteDataTypes
= [NSSet setWithArray:@[
                        WKWebsiteDataTypeDiskCache,
                        WKWebsiteDataTypeOfflineWebApplicationCache,
                        WKWebsiteDataTypeMemoryCache,
                        WKWebsiteDataTypeLocalStorage,
                        WKWebsiteDataTypeCookies,
                        WKWebsiteDataTypeSessionStorage,
                        WKWebsiteDataTypeIndexedDBDatabases,
                        WKWebsiteDataTypeWebSQLDatabases
                        ]];
//// All kinds of data
//NSSet *websiteDataTypes = [WKWebsiteDataStore allWebsiteDataTypes];
//// Date from
NSDate *dateFrom = [NSDate dateWithTimeIntervalSince1970:0];
//// Execute
[[WKWebsiteDataStore defaultDataStore] removeDataOfTypes:websiteDataTypes modifiedSince:dateFrom completionHandler:^{
   // Done
}];
```

Stack Overflow [How to remove cache in WKWebview?](http://stackoverflow.com/a/32491271/3283039)


## Scroll rate bug on iOS 9

On iOS 8, the below code works fine, it can scroll with more inertia.

`webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;`

As for iOS 9, this code is meaningless, without setting the rate value within UIScrollView delegate `scrollViewWillBeginDragging`.

```objc
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView
{
    scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
}
```

See Stack Overflow: [Cannot change WKWebView's scroll rate on iOS 9](http://stackoverflow.com/questions/31369538/cannot-change-wkwebviews-scroll-rate-on-ios-9-beta)


## Cannot disable long press link menu

CSS: `-webkit-touch-callout: none;` and JavaScript: `document.documentElement.style.webkitTouchCallout='none';` won't work.

Note: This bug was fixed in iOS 8.2.

## Sometimes capturing `WKWebView` fails

Sometimes capturing a screenshot of `WKWebView` itself failed, try to capture `WKWebView`'s `scrollView` property instead.

Otherwise, if you are not afraid of using private API, try [lemonmojo/WKWebView-Screenshot](https://github.com/lemonmojo/WKWebView-Screenshot).


## Xcode 6.1 and above doesn't indicate precise memory usage for WKWebView

As of Xcode 6.1, it indicates lower memory usage than is actually used.


## `window.webkit.messageHandlers` won't work on some websites
Some websites somehow override JavaScript's `window.webkit`. To prevent this issue, you should cache this to a variable before a website's content starts loading. `WKUserScriptInjectionTimeAtDocumentStart` can help you.


## Cookie saving sometimes failed
Are cookies synced between `NSHTTPCookie` and `WKWebView` at some point?

Thanks to @winzig, he gives me information: "Cookie discussion / ajax #3"

See this Stack Overflow question: [Can I set the cookies to be used by a WKWebView?](http://stackoverflow.com/questions/26573137/can-i-set-the-cookies-to-be-used-by-a-wkwebview)

At `WKWebView` initialization time, it can set cookies to both cookie management areas without waiting for the areas to be synced.


## `WKWebView`'s `backForwardList` is readonly
I want `WKWebView` to restore its paging history.

## Hard to coexist with `UIWebView` on iOS 7 and below
Before some person tried to submit thier app for both iOS 7 and iOS 8 using `UIWebView` and `WKWebView`, the submission was rejected right at the time.

See this issue [Cannot coexist with UIWebView on iOS 7 and below](https://github.com/ShingoFukuyama/WKWebViewTips/issues/2)


## Links
[Naituw/WBWebViewConsole](https://github.com/Naituw/WBWebViewConsole)
"WBWebViewConsole is an In-App debug console for your UIWebView && WKWebView"


## Conclusion
As you can see, `WKWebView` still looks hard to use and UIWebView looks easy.

However, Apple announced to developers:

> Starting February 1, 2015, new iOS apps uploaded to the App Store must include 64-bit support and be built with the iOS 8 SDK, included in Xcode 6 or later.

It is possible Apple will make `UIWebView` deprecated. See [64-bit and iOS 8 Requirements for New Apps](https://developer.apple.com/news/?id=10202014a).



If you're curious how `WKWebView` works for web browser apps, try my Ohajiki Web Browser.
http://en.ohajiki.ios-web.com

