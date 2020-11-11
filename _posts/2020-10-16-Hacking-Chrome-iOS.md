---
title: iOS Chromium Overlooked and Underappreciated
author: Alison Huffman
date: 2020-10-16 09:00:00 -0700
categories: [Vulnerabilities, Exploit]
tags: [iOS]
math: true
twitter_image: /assets/img/blog_img/ios/webframes.jpg
author_twitter: ohnonull
---


A Bold New Frontier
===================================
As part of Microsoft's Edge's move to using Chromium as the backbone of our browsers, we are updating all our browser product lines, including the iOS version. As security engineers on the Microsoft Edge team, one of our responsibilities is reviewing code that could potentially impact the security of the browser. This includes any changes to areas that have been prone to bugs in the past such as the addition of new Mojo interfaces.

When the security team first met with the engineers in charge of making iOS Edge a reality, we were unsure on what recommendations to make. Nobody on the team had experience looking for bugs in Chromium on iOS. It's just a wrapper around WebKit right? The developers cannot possibly add any bugs we need to worry about...right?

Unfortunately, the answer is no they can.

In the first part of this blog post series, I will be providing a quick introduction to the iOS Chromium browser and the potential attack surface of the JavaScript interprocess communication (IPC) provided by [`WKWebView`](https://developer.apple.com/documentation/webkit/wkwebview). I will also talk about a UXSS bug that I found while exploring this area of the code.

Don't Blink
===================================
<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/ios/angel.png" alt="A weeping angel from the Doctor Who series" align="right"> Unlike the desktop version of the Chromium browser, which uses V8 for its JavaScript engine and Blink for its rendering and WebAPI support, the iOS version of the Chromium does not. In fact, most likely all third-party browsers on iOS use the built-in `WKWebView` API to add web support to their application. Chromium is no exception. The reason for this is the code signing restrictions on the iOS platform and Apple's unwillingness to allow developers to submit apps with the dynamic-codesigning entitlement enabled. This entitlement in particular is what would allow applications to map RWX memory. This capability would allow the implementation of an efficient JavaScript engine. Rather than having Chromium suffer from subpar performance on the platform it's better to just use a `WKWebView` like every other browser.

IPC
===================================

One unfortunate side effect of using a `WKWebView` versus your own rendering and JavaScript engine, is you can no longer make direct modifications to the underlying code. This means you cannot directly expose new native features to websites. The `WKWebView` API remedies this through the [`WKUserContentController`](https://developer.apple.com/documentation/webkit/wkusercontentcontroller) object.  This object can be passed in along during the creation of your `WKWebView` object through a [`WKWebViewConfiguration`](https://developer.apple.com/documentation/webkit/wkwebviewconfiguration) object.

This object can be used to not only inject JavaScript **directly** into a webpage but also expose native or objective-c code through JavaScript. These JavaScript interfaces are exposed through the `window.webkit.messageHandlers` object and support calls through a postMessage interface.  At the time of writing this blog post, there are four message handlers available through every webpage. The easiest way to find these is to grep the code for "setScriptMessageHandler".

* `window.webkit.messageHandlers.FrameBecameAvailable.postMessage(...)`
* `window.webkit.messageHandlers.FrameBecameUnavailable.postMessage(...)`
* `window.webkit.messageHandlers.crwebinvoke.postMessage(...)`
* `window.webkit.messageHandlers.FindElementResultHandler.postMessage(...)`


**Frame Handlers**

Let's discuss the first two message handlers as a pair. These are the handlers that control the creation and destruction of the [`WebFrameImpl`](https://source.chromium.org/chromium/chromium/src/+/master:ios/web/js_messaging/web_frame_impl.mm) object. This object's sole responsibility is to manage the IPC communications between the browser process and the frames within a page. Internally, these objects are managed by the [`WebFramesManagerImpl`](https://source.chromium.org/chromium/chromium/src/+/master:ios/web/js_messaging/web_frames_manager_impl.mm) which is owned by a [`WebStateImpl`](https://source.chromium.org/chromium/chromium/src/+/master:ios/web/web_state/web_state_impl.mm). The `WebStateImpl` object is the main object that represents a tab and its equivalent on the desktop is the [`WebContents`](https://source.chromium.org/chromium/chromium/src/+/master:content/public/browser/web_contents.h) object.

<div style="text-align:center">
	<img alt="A picture of a couple with the Chromium logo for heads infinitely recursing into a picture frame." src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/ios/webframes.jpg">
	<div style="margin:10px 0px; font-weight: bold;">A happy WebFrameImpl family</div>
</div>


Unlike the desktop version of Chromium, where objects like the `RenderFrameHost` are tightly coupled with the lifetime of frames, the lifetime of the `WebFrameImpl` objects are completely controlled by JavaScript. If a webpage is randomly calling `FrameBecameAvailable` or `FrameBecameUnavailable`, a frame may not have a corresponding `WebFrameImpl` object or it may have ten. I will mention that I have not noticed any memory safety issues caused by this API being exposed and usable in this way.

These handlers are normally invoked through the code in the [//ios/web/js_message/resources](https://source.chromium.org/chromium/chromium/src/+/master:ios/web/js_messaging/resources/) directory.

To create a `WebFrameImpl` object the following snippet is all that's needed.

```javascript
window.webkit.messageHandlers.FrameBecameAvailable.postMessage({
  "crwFrameId": "a8ffcfb8a97236ee42bff55184584ba8", /* frame identifier */
  "crwFrameKey": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=", /* sub-frame message encryption key */
  "crwFrameLastReceivedMessageId": 0 /* reply message ID counter */
});
```

To destroy a `WebFrameImpl` object the following snippet will suffice.

```javascript
window.webkit.messageHandlers.FrameBecameUnavailable.postMessage("a8ffcfb8a97236ee42bff55184584ba8")
```

In the snippet above, the only thing needed to destroy a frame's `WebFrameImpl` object is its identifier. These identifiers, with the exception of the main frame's, are not secret.

This is because if the native side of the browser wants to execute JavaScript inside a subframe, it will first route the message through the main frame. This is done by calling the [`WebFrameImpl::CallJavaScriptFunction`](https://source.chromium.org/chromium/chromium/src/+/ff136563a019342bb27060535e18cce9d57f7099:ios/web/js_messaging/web_frame_impl.mm;l=121) method. This method uses the target frame's encryption key to encrypt the function name and parameters and then injects a call to [`__gCrWeb.message.routeMessage`](https://source.chromium.org/chromium/chromium/src/+/ff136563a019342bb27060535e18cce9d57f7099:ios/web/js_messaging/resources/message.js;l=375) into the page's main frame. The receiving frame then checks the target frame field in the message against its own frame ID. If it does not match, it loops through all child frames and re-posts the message. Eventually the message will make its way to the correct child frame and it will decrypt the message and invoke the desired function. The reason sub-frame frame IDs are not secret, is because any child frame can install a message listener and examine the target frame ID field of the messages that pass through it.

The way these `routeMessage` calls are constructed combined with an oversight in the `FrameBecameAvailable` handler actually led to a fairly serious bug. When injecting the `routeMessage` call into the main frame the following code was used.

```c++
std::string script =
    base::StringPrintf("__gCrWeb.message.routeMessage(%s, %s, '%s')",
                       encrypted_message_json.c_str(),
                       encrypted_function_json.c_str(), frame_id_.c_str());
GetWebState()->ExecuteJavaScript(base::UTF8ToUTF16(script));
 ```

As one can see the script is constructed using a format string using `encrypted_message_json`, `encrypted_function_json`, and the `frame_id_` member of the `WebFrameImpl`. Originally, the `frame_id_` member had no limitations on the name you could provide. Due to this, a child frame could execute the following code:

```javascript
window.webkit.messageHandlers.FrameBecameAvailable.postMessage({
  'crwFrameId': "'foobar');alert(window['origin']+'",
  'crwFrameKey': 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
  'crwFrameLastReceivedMessageId': 0
});
```

This would cause a new `WebFrameImpl` to be created with a frame ID of `foobar');alert(window['origin']+'`. Upon the creation of the frame, the browser would try to route a message to it. Specifically, it would try to get the form information from the frame for autofill. Since JavaScript is injected into the main frame this would end in the following code running in the main frame:

```javascript
__gCrWeb.message.routeMessage('XXXXXX', 'XXXXXX', 'foobar');alert(window['origin']+'')
```

The impact of this bug is it enabled a UXSS from a child frame, such as one hosting an advertisement, and allows the frame to run arbitrary JavaScript in the webpage hosting the frame. This bug was [reported](https://bugs.chromium.org/p/chromium/issues/detail?id=1098606) and promptly had a [fix](https://chromium.googlesource.com/chromium/src.git/+/37bccc228e1089bb07e37c9137bd03437b7c4de7) made within two days. The fix itself was simple and ensures that frame IDs only contain hex digits (0-9 and A-F).

**crwebinvoke**

The `crwebinvoke` handler is used for handling messages tied to the `WebState` object of a tab. These handlers are added by calling `WebStateImpl::AddScriptCommandCallBack`  and the callbacks themselves have the following signature.

```objc
const web::WebState::ScriptCommandCallback callback = base::BindRepeating(
  ^(const base::DictionaryValue& content, const GURL& mainDocumentURL,
    bool userInteracting, web::WebFrame* senderFrame) {
  	...
}
```

Something interesting about the fourth argument of `ScriptCommandCallback` `senderFrame` is that it is totally dependent on what ID is passed in the `crwFrameId` field of the message. This combined with the fact that frame ID's of child frames can be leaked could be a recipe for bugs in the future. Currently, I have not noticed this behavior causing any issues. It is worth noting that many callbacks will do something similar to `if(sender_frame->IsMainFrame())` before doing certain operations. This means that if you can somehow leak the 128-bit value that represents the main frame ID you would be able to impersonate the main frame. I fortunately did not notice any ways to do this.

Just as an example of this interface, and unrelated to any security issues, is the code that handles the [`window.print`](https://source.chromium.org/chromium/chromium/src/+/master:ios/chrome/browser/web/print_tab_helper.mm) call.

```javascript
let msg = {}
msg["command"] = "print.print"
let message = {'crwCommand': msg,
               'crwFrameId': __gCrWeb.message.getFrameId()}
window.webkit.messageHandlers.crwebinvoke.postMessage(message);
```

If you execute the above code in a webpage, then the print dialogue will open.

Another functionality that the `crwebinvoke` interface provides is a channel to send results from injected JavaScript calls. Whenever a method is invoked through `routeMessage` the browser internally adds a callback handler to a [list](https://source.chromium.org/chromium/chromium/src/+/91d5a91b0fbf1a13d17d9745696d719ff01056a2:ios/web/js_messaging/web_frame_impl.mm;l=181). These handlers can be reached by sending a `frameMessaging_<FRAME ID>.reply` [command](https://source.chromium.org/chromium/chromium/src/+/cab7381283bebbc4d3fe64b3dfa81d5b3e418cdd:ios/web/js_messaging/resources/message.js;l=205). The string `FRAME ID` should be replaced with the frame ID of the frame that was originally sent the request. Since the `WKWebView` does not expose direct access to the DOM of the webpage this functionality is typically used to retrieve information about the contents of the page.

One potential issue with this implementation detail is that all that is needed to forge replies is the knowledge of the frame ID that sent the request and the message ID of the request. The message ID as it turns out is just an incrementing 32-bit integer from the `crwFrameLastReceivedMessageId` parameter of when the frame was created. For new frames, this message ID starts at zero. This means that any child frame can theoretically forge the callbacks of any other child frame. However, I have not noticed any exploitable issues with subframe messages yet.

**FindElementResultHandler**

Finally, the `FindElementResultHandler` handler is responsible for informing the browser of the HTML element that is being touched. Whenever a webpage is touched the browser will inject a call to  `__gCrWeb.findElementAtPoint` in the main frame.  When `findElementAtPoint` detects the touched element is in a sub-frame of a differing origin, a message is posted to the child frame so it can handle informing the browser. This is used to generate the long-press context menu and is implemented in the [`CRWContextMenuController`](https://source.chromium.org/chromium/chromium/src/+/master:ios/web/web_state/ui/crw_context_menu_controller.mm) class.


Closing
===================================
I hope by outlining the available attack surface within the IPC of the browser other researchers can give it the attention it deserves. While most of the concepts discussed in this article pertain specifically to Chromium based iOS browsers, all `WKWebView` consumers have access to the `messageHandler` functionality. It is a great first place to look when doing an audit of a `WKWebView` based browser.  While looking at the iOS version of Chromium, I discovered multiple issues, some of which are still being fixed. Once the issues are fixed, I hope to post an entry about them on this blog.
