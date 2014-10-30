This is what I learned about WKWebView: Apple's new WebKit API debuted on iOS 8.
As of this writing, latest iOS version is iOS 8.1.0.

## file:/// don't work without tmp directory
Only tmp directory access with file protocol has been allowed from iOS 8.0.2.
You can see what directory access allowed with [GitHub shazron / WKWebViewFIleUrlTest](https://github.com/shazron/WKWebViewFIleUrlTest)

## Can't handle in Storyboard and Interface Builder
You need to set WKWebView and NSLayoutConstraint programmatically.

## HTML a tag with target="_blank" won't respond
stackoverflow - [Why is WKWebView not opening links with target=“_blank”](http://stackoverflow.com/questions/25713069/why-is-wkwebview-not-opening-links-with-target-blank)

## URL Scheme and App Store link won't work

#### For example

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

## alert, confirm, prompt from JavaScript needs to set WKUIDelegate methods
If you want to show dialogues, you have to set the following methods:
`webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:`
`webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:`
`webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:`
Here is how to set those: http://qiita.com/ShingoFukuyama/items/5d97e6c62e3813d1ae98

## Basic/Digest/etc authentication input dialogues need to set a WKNavigationDelegate method
If you want to present authentication challenge to user, you have to set the method below:
`webView:didReceiveAuthenticationChallenge:completionHandler:`

#### For example:

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
        
    } else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
}
```

## Cookie sharing between multiple WKWebViews

#### For example

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

stackoverflow - http://stackoverflow.com/questions/25797972/cookie-sharing-between-multiple-wkwebviews

## NSCachedURLResponse won't work
UIWebView can filter Ads URLs and cache to read website offline using NSURLCache and NSCachedURLResponse.
But WKWebView can't.

## Cookie, Cache, Credential, WebKit data cannot easily delete
After much cut-and-try, I've reached the following conclusion:

1. Use NSURLCache and NSHTTPCookie to delete cookies and caches in the same way as you used to do on UIWebView. If you use WKProccessPool, re-initialize it.
2. Delete `Cookies`, `Caches`, `WebKit` subdirectory in Library directory.
3. Delete all WKWebViews

## Cannot disable long press link menu
CSS: `-webkit-touch-callout: none;` and JavaScript: `document.documentElement.style.webkitTouchCallout='none';` won't work.

## Sometimes capturing WKWebView failed
Capture WKWebView's scrollView property instead of WKWebView itself.

## Xcode 6.1 don't indicate precise memory usage for WKWebView
As of Xcode 6.1, it indicate lower memory usage than actually is.

## window.webkit.messageHandlers won't work on some websites
Some websites somehow override JavaScript's `window.webkit`. To prevent this issue, you should cache this to a variable before a website's content starts loading. `WKUserScriptInjectionTimeAtDocumentStart` would help you.

## Cookie saving sometimes failed
Are cookies synched between NSHTTPCookie and WKWebView at some point?


## WKWebView's backForwardList is readonly
I want WKWebView to restore its pageing history.

## Cannot coexist with UIWebView
If you try to submit an app using UIWebView and WKWebView, the submission would be failed. In other words, you have to stop supporting iOS 7 and before iOS versions to use WKWebView.

## IMO
As you can see, WKWebView still hard to use and UIWebView looks easy.
However, Apple announced to developers 'Starting February 1, 2015, new iOS apps uploaded to the App Store must include 64-bit support and be built with the iOS 8 SDK, included in Xcode 6 or later'. It is possible Apple make UIWebView deprecated.
[64-bit and iOS 8 Requirements for New Apps](https://developer.apple.com/news/?id=10202014a)



If you curious how WKWebView for Web Browser App works, would you mind to try my Ohajiki Web Browser?
http://en.ohajiki.ios-web.com

