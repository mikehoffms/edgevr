---
title: Deep Dive into Site Isolation (Part 1)
author: Jun Kokatsu
date: 2020-11-10 10:00:00 -0800
categories: [Vulnerabilities]
tags: [Site Isolation]
math: true
author_twitter: shhnjk
---

Back in [2018](https://security.googleblog.com/2018/07/mitigating-spectre-with-site-isolation.html), Chrome enabled [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation) by default, which mitigates attacks such as UXSS and [Spectre](https://v8.dev/blog/spectre). At the time, I was actively participating in the [Chrome Vulnerability Reward Program](https://www.google.com/about/appsecurity/chrome-rewards/), and I was able to find 10+ bugs in Site Isolation, resulting in $32k rewards.

In this blog post series, I will explain how Site Isolation and related security features work, as well as explain the bugs found in those security features that have since been patched.

# Methodology
In my approach for bug hunting for Chrome, I would usually start with manual testing rather than code audit. This is because the Chrome team is generally good at code reviews. So I think that most of the logical bugs that slip through their code reviews are difficult to find by code audit. Therefore, I followed the same methodology when I began looking at Site Isolation.

# What is Site Isolation?
[Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation) is a security feature that separates web pages from each Site to its own process. With Site Isolation, the boundary of a Site is aligned with OS-level process isolation, instead of in-process logical isolation, such as same-origin policy.

_Site_ is defined as <span style="color:red">Scheme</span> and <span style="color:DodgerBlue">[eTLD](https://web.dev/same-site-same-origin/#:~:text=effective%20TLDs)+1</span> (also known as [Schemeful same-site](https://web.dev/same-site-same-origin/#%22schemeful-same-site%22)).

<span style="color:red">https</span>://www.<span style="color:DodgerBlue">microsoft.com</span>:443/

The definition of _Site_ is broader than _Origin_, which is <span style="color:red">Scheme</span>, <span style="color:DodgerBlue">Host</span>, and <span style="color:green">Port</span>.

<span style="color:red">https</span>://<span style="color:DodgerBlue">www.microsoft.com</span>:<span style="color:green">443</span>/

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/samesite.png" width="370" height="300"><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/crosssite.png" width="370" height="300">

However, not all cases fit into the above definition of a _Site_. So I started testing the following edge-cases to see how Site Isolation behaves in each case 😊

# URL without a domain name
URLs are not required to contain a domain name (e.g. IP address). In this case, Site Isolation falls back to origin comparison for process isolation.

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/samesite-ip.png" width="370" height="300"><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/crosssite-ip.png" width="370" height="300">

# File URL
Local files can be rendered into a browser tab with the file scheme. Currently, Site Isolation treats any URL with the file scheme as originating from the same-site.

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/samesite-file.png" width="370" height="300"><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/crosssite-file.png" width="370" height="300">

# Data URL
While a Data URL loaded on the top frame is always isolated to its own process, a Data URL loaded inside an iframe will inherit the Site from the navigation initiator (even though the origin remains an [opaque origin](https://html.spec.whatwg.org/multipage/origin.html#concept-origin-opaque)). If you are old enough, you would remember similar concept where Data URL inside an iframe used to [inherit the origin in Firefox](https://blog.mozilla.org/security/2017/10/04/treating-data-urls-unique-origins-firefox-57/) 😊

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/samesite-data.png" width="370" height="300"><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/crosssite-data.gif" width="370" height="300">

As you can see in above images, even though both examples result in microsoft.com embedding a Data URL iframe, the cross-site case remains process isolated because the navigation initiator was evil.example.

## A bug in Data URL Site inheritance
What happens if a browser or a tab is restored from the local cache? Will Site Isolation still remember the navigation initiator of a Data URL, or will it forget?

It turns out that Site Isolation can’t remember the navigation initiator after a browser or a tab restore. And when a tab was restored from local cache, Site Isolation used to blindly put the Data URL within the same-site of the parent frame. An attacker could have triggered a [browser crash](https://bugs.chromium.org/p/chromium/issues/detail?id=935050) and then restoring the browser would cause a Site Isolation bypass 😊

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/data-restore-bug.gif">

[This bug](https://bugs.chromium.org/p/chromium/issues/detail?id=863069) was fixed by isolating the Data URL into a different process when the page is restored from the local cache.

# Blob URL with opaque origin
When a Blob URL is created from an opaque origin (e.g. Data URL, sandboxed iframe, or File URL), the Blob URL would look like “blob:null/[GUID]”. 

Because this is a [URL without a domain name](#url-without-a-domain-name), Site Isolation used to fall back to origin comparison. However, this was a [bug](https://bugs.chromium.org/p/chromium/issues/detail?id=863623) because a Blob URL with an opaque origin can be created from a different Site. And the origin comparison wouldn’t be enough because the origin will always be “blob:null”. Therefore, this URL requires the comparison of both origin and path for process isolation.

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/samesite-blob.png" width="370" height="300"><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/crosssite-blob.png" width="370" height="300">

# Testing process isolation logic
I was able to find some bugs in the process isolation logic of Site Isolation using tools such as the Chromium Task Manager (Shift + Esc on Windows) and [FramesExplorer](https://chrome.google.com/webstore/detail/framesexplorer/imijdbpfemdegalijeojlkhiamfcgklp), which helps identify those frames which share a process (additionally, you can also use [chrome://process-internals/#web-contents](chrome://process-internals/#web-contents)).

# How does Site Isolation mitigate UXSS?
Historically, [Most of UXSS bugs](https://research.google/pubs/pub48028.pdf) were achieved by bypassing the same-origin policy check that is mostly implemented in a renderer process. In other words, once you can bypass the same-origin policy check, all the cross-site data was available within the renderer process. Therefore, JS code was able to get window, document, or any other cross-origin object references.

Site Isolation fundamentally changed this by process isolation. Even if you can bypass the same-origin policy check, other site's data won’t be available in the same process.

Additionally, a UXSS vector where a cross-origin window is navigated to a JavaScript URL isn’t an issue either. This is because JavaScript URL navigation should only succeed if 2 web pages are of the same origin. Therefore, navigation to JavaScript URLs can be handled within the renderer process (which should host any same-origin pages that have window reference), and any JavaScript URL navigation request that goes up to the browser process can be safely ignored.
Of course, if you forget to ignore a JavaScript URL in the browser process, that's a [bug](https://bugs.chromium.org/p/chromium/issues/detail?id=962500) (which was found by [@SecurityMB](https://twitter.com/SecurityMB))😉

# Process Isolation isn't enough
Site Isolation helps mitigate UXSS, however process isolation isn’t enough to protect all cross-site data from leaking.

The easiest example of this is the [Spectre](https://v8.dev/blog/spectre) attack. With Spectre, an attacker can [read a process’s entire address space](https://v8.dev/blog/spectre#site-isolation&:~:text=read%20a%20process%E2%80%99s%20entire%20address%20space). While process isolation helps isolate web pages, subresources such as image, audio, video, etc, are not process isolated.

Therefore, without another mitigation, an attacker can read an arbitrary cross-site page by embedding that page using _img_ tag.

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/spectre_read.PNG" width="750" height="250">

# Cross-Origin Read Blocking
[Cross-Origin Read Blocking](https://www.chromium.org/Home/chromium-security/corb-for-developers) (CORB) mitigates the risk of exposing sensitive cross-origin data by checking MIME type of responses in cross-origin subresources. If the MIME type of cross-origin subresouces is in a deny list (e.g. HTML, XML, JSON, etc), the response will not be sent to the renderer process by default, and therefore it won't be readable using a Spectre attack.

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/CORB.PNG" width="750" height="250">

# CORB Bypasses
There have been multiple CORB bypasses in the past.
1. [CORB bypass in Workers](https://bugs.chromium.org/p/chromium/issues/detail?id=933777) by [@_tsuro](https://twitter.com/_tsuro)
2. [CORB bypass in WebSocket](https://bugs.chromium.org/p/chromium/issues/detail?id=944619) by [@piochu](https://twitter.com/piochu)
3. [CORB bypass in AppCache](https://bugs.chromium.org/p/chromium/issues/detail?id=949384)

Fundamentally, CORB can be bypassed if the URL was fetched using [URLLoader](https://source.chromium.org/chromium/chromium/src/+/master:services/network/public/mojom/network_context.mojom;l=609;drc=0e22286ee95d2bd711f81d73f1178343fbacc890) with [CORB disabled](https://source.chromium.org/search?q=%22is_corb_enabled%20%3D%20false%22), and the response was leaked to a cross-site web renderer process. 

For example, I was able to bypass CORB using AppCache because the _URLLoader_ that was used to download resources for caching had CORB disabled. I knew that AppCache used to allow the caching of cross-origin HTML files, so I figured this might bypass CORB, and it did 😊

It's also important to note that some renderer processes can bypass CORB by design (e.g. extension renderer process).

# Cross-Origin Resource Policy
While CORB can protect many sensitive resources by default, some resources such as images can be embedded across sites on the Web. Therefore, CORB can't protect such resources from entering a cross-origin page by default.

[Cross-Origin Resource Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cross-Origin_Resource_Policy_(CORP)) (CORP) allows developers to indicate if a specific resource can be embedded to _same-origin_, _same-site_, or _cross-origin_ pages. This indication allows a browser to protect resources that can't be protected by CORB by default. 

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/site_isolation/CORP.PNG" width="750" height="250">

With CORP, websites can protect sensitive resources from attacks such as Spectre, XSSI, [SOP bypasses specific to subresources](https://speakerdeck.com/shhnjk/logically-bypassing-browser-security-boundaries?slide=24), and so on. In other words, sensitive resources that lack a CORP header can be read using a Spectre attack (unless the resources are protected by CORB).

# How to test CORB
If you have an idea on how to bypass CORB where you can load a subresource (e.g. [CORB bypass](#corb-bypasses) via AppCache above), try to load that subresource with following response headers:
```
Content-Type: text/html
X-Content-Type-Options: nosniff
```
CORB should block such a cross-origin subresource, and therefore if the subresource was correctly loaded, that indicates a CORB bypass.

If you have an idea where subresources can't be loaded (e.g. [CORB bypass](#corb-bypasses) via WebSocket above), try using [WinDbg](https://www.chromium.org/developers/how-tos/debugging-on-windows/windbg-help)'s [_!address_](https://learn.microsoft.com/windows-hardware/drivers/debugger/-address#:~:text=Quotes%20in%20the%20command,for%20the%20string%20%22pad%22) command to find if the string you are looking for has actually entered into the renderer process by searching that string in a heap.
```
!address /f:Heap /c:"s -a %1 %2 \"secret\"" 
```
This searches string _**secret**_ in heap memory of the renderer process. If you are looking for a unicode string instead of an ascii string, you can change _-a_ to _-u_ (more details in the [search memory](https://learn.microsoft.com/windows-hardware/drivers/debugger/s--search-memory-) section).

# What’s next?
With Site Isolation, CORB, and CORP, we can mitigate many attacks from UXSS to Spectre along with many other client-side attacks, which allow an attacker to read cross-site information. However, there are attacks which can fully compromise a renderer process. The next part will focus on how Site Isolation was further tightened to mitigate such an attacker from obtaining cross-site information.
