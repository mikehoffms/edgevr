---
title: Yet another RenderFrameHostImpl UAF
author: Lucas P.
date: 2021-03-02 08:00:00 -0800
categories: [Vulnerabilities, Exploit]
tags: [Android]
math: true
---

# Introduction

Back in 2020 while reviewing Chromium code, I found issue [1068395], a
Use-After-Free in Browser Process that can be used to escape the Chromium
sandbox on Android Devices. [This][uaf-bug-1] [is][uaf-bug-2] [an][uaf-bug-3]
[interesting][uaf-bug-4] vulnerability as it's a bug pattern [that][uaf-bug-5]
keeps [happening][uaf-bug-6] in the Chromium codebase.

Having a good understanding of this pattern and how an attacker can
exploit it is a good exercise to gain knowledge as well as inspiration
in what to look for when reviewing code and writing new fuzzers. More
importantly this can help us learn how we can mitigate such bug patterns.

Today, we will explore how issue [1068395] can be exploited assuming a compromised
Renderer Process (using a vulnerability like [1126249]).

[1068395]: https://bugs.chromium.org/p/chromium/issues/detail?id=1068395
[uaf-bug-1]: https://bugs.chromium.org/p/chromium/issues/detail?id=1134480
[uaf-bug-2]: https://bugs.chromium.org/p/chromium/issues/detail?id=1101509
[uaf-bug-3]: https://bugs.chromium.org/p/chromium/issues/detail?id=1128270
[uaf-bug-4]: https://bugs.chromium.org/p/chromium/issues/detail?id=1122917
[uaf-bug-5]: https://bugs.chromium.org/p/chromium/issues/detail?id=1078671
[uaf-bug-6]: https://bugs.chromium.org/p/chromium/issues/detail?id=1106342
[1126249]: https://bugs.chromium.org/p/chromium/issues/detail?id=1126249

# What Is a RenderFrameHost?

Whenever a website is navigated to, the Browser Process will spawn a new
Renderer Process. This process will parse the website’s content, such as
JavaScript, HTML, and CSS and display it on its main frame. To track the
main frame and communicate with it, the Browser Process will instantiate a
[`RenderFrameHostImpl` (RFH) object] to represent the Renderer's main frame.

Complicating things further, a website may have multiple "child-frames"
(a.k.a iframes) that will embed another page context inside the main frame which
can be created and destroyed at any moment by JavaScript. If the embedded origin
is the same as the main frame, the Renderer Process will create a "frame" object
and track it using a frame-tree data structure. The Browser Process will mirror
this behavior and create a new [RFH object for each new child frame]. However,
if the contexts have a different origin, the Browser Process will spawn a new
[Renderer Process due to site isolation].

For us, the behavior described previously may be read as: We control the RFH's
object creation and destruction (or its lifetime) from JavaScript! Can we
leverage such behavior to find security vulnerabilities?

<!-- TODO(alison): please repoint these links to commits with line numbers -->
[`RenderFrameHostImpl` (RFH) object]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/renderer_host/render_frame_host_impl.h;l=252;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[RFH object for each new child frame]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/renderer_host/frame_tree_node.h;l=53;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[Renderer Process due to site isolation]: https://www.chromium.org/developers/design-documents/site-isolation

### RenderFrameHost and Mojo interfaces

Nowadays, modern web-browsers are implemented with a multi-process architecture
in mind. In this model, we have untrusted content parsed in a very
restrictive/locked-down process (a.k.a a "sandboxed process"). To provide
resource access to such a locked-down process, we have a "broker process". In
our case, this is the “browser process”. The browser process can provide access
to these restricted resources via a
[mechanism called Inter-Process-Communication (IPC)](https://en.wikipedia.org/wiki/Inter-process_communication).

Chromium has two IPC mechanisms,
[the old-IPC/Legacy IPC](https://chromium.googlesource.com/chromium/src/+/master/docs/mojo_ipc_conversion.md)
and
[Mojo IPC](https://chromium.googlesource.com/chromium/src/+/master/mojo/README.md).
Nowadays, most features that want to expose resources to the Renderer Process
(untrusted/sandboxed process) do it by
[using a Mojo interface](https://chromium.googlesource.com/chromium/src/+/master/docs/mojo_and_services.md).
These interfaces are described using a mojom file, for example (from
[mojo\_and\_services.md]):

```c++
interface PingResponder {
  // Receives a "Ping" and responds with a random integer.
  Ping() => (int32 random);
};
```

A Mojo interface is usually bound per-frame, therefore, every time an iframe is
created and you request to bind to new a Mojo interface, it may end up
allocating a new Mojo interface object in Browser Process (or bind into an
existing one). If you are curious about what interfaces are exposed to an
iframe, you can check the [browser_interface_binders.cc][Browser-Interface-Binders].
There is a "trick" here, as explained by [`BrowserInterfaceBroker`][BrowserInterfaceBroker], different
"execution types" (a.k.a iframe/document, service workers and so on), may have a
different binder function (thus, different set of interfaces exposed), it can be
observed in [`PopulateServiceWorkerBinders`][PopulateServiceWorkerBinders] and
[`PopulateFrameBinders`][PopulateFrameBinders]. (sidenote: Monitoring the commit
changes in [browser_interface_binders.cc][Browser-Interface-Binders] is a nice method to find new Mojo
Interfaces!).

Many objects accessible over Mojo don't need access to the web page itself and
are there to just facilitate access to privileged operations not allowed in the
sandbox. However, there are situations that a Mojo interface object may require
to access to the [RFH object that has instantiated it], like accessing its RFH's
`WebContentsImpl` object, accessing its `RenderFrameProcess` object and so on.

One way to accomplish this is by providing a raw pointer to the RFH that has
instantiated the interface in the Mojo interface object constructor. The
constructor can then store this pointer as a class member. You can observe such
behavior in the [`SensorProviderProxyImpl`][SensorProviderProxyImpl]:

<a id="code-note-1"></a>
```c++
SensorProviderProxyImpl::SensorProviderProxyImpl(
    PermissionControllerImpl* permission_controller,
    RenderFrameHost* render_frame_host)
    : permission_controller_(permission_controller),
      render_frame_host_(render_frame_host) { // [1]

  DCHECK(permission_controller);
  DCHECK(render_frame_host);
}
```

As you can see at [[1]](#code-note-1),
[`SensorProviderProxyImpl`][SensorProviderProxyImpl] will store the raw pointer
for the RFH that has instantiated it as a member. Now, there is a question, can
we guarantee that the Mojo interface will not outlive (stay alive longer than)
RFH object? The answer can be found by checking how the Mojo interface object
gets created. Let’s look at the code below.

<a id="code-note-2"></a>
```c++
void RenderFrameHostImpl::GetSensorProvider(
    mojo::PendingReceiver<device::mojom::SensorProvider> receiver) {
  if (!sensor_provider_proxy_) {
    sensor_provider_proxy_ = std::make_unique<SensorProviderProxyImpl>( // [2]
        PermissionControllerImpl::FromBrowserContext(
            GetProcess()->GetBrowserContext()),
        this);
  }
  sensor_provider_proxy_->Bind(std::move(receiver));
}
```

The `SensorProvider` Mojo interface object is a member variable in the
`RenderFrameHostImpl` class [[2]](#code-note-2). If the `sensor_provider_proxy_`
has not been initialized yet, it'll instantiate a [`std::unique_ptr`] for it.
So, we can guarantee that [`SensorProviderProxyImpl`][SensorProviderProxyImpl]
object will be destroyed once the RFH object gets destroyed as their lifetimes
are tied to each other!

However, Chromium is a complex code-base and things aren't always that easy;
there are other ways in which Mojo interface objects may be created. For example,
one may be instantiated by using [`Mojo::MakeSelfOwnedReceiver`].
[The documentation][MakeSelfOwnedReceiver docs] states: "A self-owned receiver exists
as a standalone object which owns its interface implementation and automatically
cleans itself up when its bound interface endpoint detects an error."

In other words, the lifetime for the Mojo interface object is tied with its mojo
connection: so, if the mojo connection stays alive, the Mojo interface object
will stay alive as well (more details in [here][Mojo interface more details]).
This means both sides of the mojo connection (Browser and Renderer Process)
control the object lifetime; this is explained well in
["Virtually Unlimited Memory: Escaping the Chrome Sandbox"][gpz-virtually-unlimited-memory]
by Mark Brand).

That also means that we can have a situation where UI thread will destroy the
RFH object, and the Mojo connection is still alive (as it is self-owned) and
processing mojo messages until the bind detects an error or that it was closed.
Thus, if during this time-window the Mojo interface object processes a message
that will access the freed RFH object, we will have a Use-After-Free (UAF)
issue.

This is an example of the problem that most of the vulnerabilities linked to in
the introduction end up exploiting. It has been exploited by many researchers,
explained in other blogposts, like [“Escaping the Chrome Sandbox"][theori_escaping_sandbox] and in CTFs
like [PlaidCTF][PlaidCTF sandbox exploit].

Chromium does have some features to mitigate such problems, and we will go
through some examples:


* [`WebContentsObserver`][WebContentsObserver]: If your Mojo interface
  implementation inherits from this class, you will be provided with a set of
  callback events (virtual methods) that may be overridden by your
  implementation. Among these callbacks, we have `RenderFrameDeleted`, which
  gets triggered every time an RFH object gets deleted.

  We can observe its use in
  [`InstalledAppProviderImpl`][InstalledAppProviderImpl]. This class was used to fix
  the vulnerability described in ["Escaping the Chrome Sandbox"][theori_escaping_sandbox].

  ```c++
  void InstalledAppProviderImpl::RenderFrameDeleted(
      RenderFrameHost* render_frame_host) {
    if (render_frame_host_ == render_frame_host) {
      render_frame_host_ = nullptr;
    }
  }
  ```

* [`FrameServiceBase`][FrameServiceBase]: This class is similar to
  `WebContentsObserver`, however, it implements all callbacks for you and
  guarantees that the implementation object gets freed as soon as the RFH object
  that created it gets deleted.

By using one of the mechanisms mentioned above, you can guarantee that the Mojo
Interfaces you own won't have Use-After-Free issues with RFH objects.

Now that we understand the complexities of Mojo interfaces and RFH, and the
problems that can arise from their mismanagement, we can start looking around to
see if we can find a vulnerability :).

[mojo_and_services.md]: https://chromium.googlesource.com/chromium/src/+/master/docs/mojo_and_services.md
[BrowserInterfaceBroker]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/browser_interface_broker_impl.h;l=32;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[PopulateServiceWorkerBinders]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/browser_interface_binders.cc;l=1114;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[PopulateFrameBinders]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/browser_interface_binders.cc;l=553;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[RFH object that has instantiated it]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/generic_sensor/sensor_provider_proxy_impl.cc;l=38;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[SensorProviderProxyImpl]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/generic_sensor/sensor_provider_proxy_impl.cc;l=38;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[`std::unique_ptr`]: https://en.cppreference.com/w/cpp/memory/unique_ptr
[`Mojo::MakeSelfOwnedReceiver`]: https://source.chromium.org/chromium/chromium/src/+/master:mojo/public/cpp/bindings/self_owned_receiver.h;l=22;drc=9db96f40a036ecfdf6ef4498f622cda70a548126
[MakeSelfOwnedReceiver docs]: https://chromium.googlesource.com/chromium/src/+/master/mojo/public/cpp/bindings/README.md
[Mojo interface more details]: https://source.chromium.org/chromium/chromium/src/+/master:mojo/public/cpp/bindings/strong_binding.h;l=45;drc=9db96f40a036ecfdf6ef4498f622cda70a548126
[theori_escaping_sandbox]: https://theori.io/research/escaping-chrome-sandbox/
[PlaidCTF sandbox exploit]: https://ctftime.org/task/11314
[WebContentsObserver]: https://source.chromium.org/chromium/chromium/src/+/master:content/public/browser/web_contents_observer.h;l=77;drc=618ce5c74c6749614a9dce25d1aa593a5389176b
[InstalledAppProviderImpl]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/installedapp/installed_app_provider_impl.cc;l=69;drc=618ce5c74c6749614a9dce25d1aa593a5389176b
[FrameServiceBase]: https://source.chromium.org/chromium/chromium/src/+/master:content/public/browser/frame_service_base.h;l=36;drc=618ce5c74c6749614a9dce25d1aa593a5389176b

# Enter SmsReceiver!
Like all normal people do at 4AM, I was using [Chromium Code Search] to read
around Chromium's source code. While looking around the commit changes for
[browser_interface_binders.cc][Browser-Interface-Binders] to check for new Mojo interfaces and other related
changes, `SmsService` caught my eye. Let's see how the Mojo interface object is
created.

<a id="code-note-3"></a>
<a id="code-note-4"></a>
```c++
void RenderFrameHostImpl::BindSmsReceiverReceiver(
    mojo::PendingReceiver<blink::mojom::SmsReceiver> receiver) {

  if (GetParent() && !GetMainFrame()->GetLastCommittedOrigin().IsSameOriginWith(
                         GetLastCommittedOrigin())) {
    mojo::ReportBadMessage("Must have the same origin as the top-level frame.");
    return;
  }

  auto* fetcher = SmsFetcher::Get(GetProcess()->GetBrowserContext(), this); // [3]
  SmsService::Create(fetcher, this, std::move(receiver)); // [4]
}
```

First, it'll call `SmsFetcher::Get` with the `BrowserContext` and `this` (RFH
object reference) as arguments [[3]](#code-note-3); we will come back for
`SmsFetcher::Get` later, but for now, all we need to know is that it'll return a
pointer to an `SmsFetcher` object. Afterwards, we will end up calling
[`SmsService::Create`] with the `SmsFetcher` object pointer and `this` (RFH object
reference) as argument [[4]](#code-note-4).

<a id="code-note-5"></a>
<a id="code-note-6"></a>
```c++
// static
void SmsService::Create(
    SmsFetcher* fetcher,
    RenderFrameHost* host,
    mojo::PendingReceiver<blink::mojom::SmsReceiver> receiver) {
  DCHECK(host);

  // SmsService owns itself. It will self-destruct when a Mojo interface
  // error occurs, the render frame host is deleted, or the render frame host
  // navigates to a new document.
  new SmsService(fetcher, host, std::move(receiver)); // [5]
}

SmsService::SmsService(
    SmsFetcher* fetcher,
    const url::Origin& origin,
    RenderFrameHost* host,
    mojo::PendingReceiver<blink::mojom::SmsReceiver> receiver)
    : FrameServiceBase(host, std::move(receiver)), // [6]
      fetcher_(fetcher),
      origin_(origin) {}
```

As the code-comments explained, the Mojo interface object owns itself
[[5]](#code-note-5). It isn't using `mojo::MakeSelfOwnedReceiver`, but
`SmsService` inherits from `FrameServiceBase` [[6]](#code-note-6), which has a
similar effect. In the `SmsService` constructor we can see it will initialize the
`FrameServiceBase` [[6]](#code-note-6) with our RFH object reference so it can
track the RFH object state.

As we have already learned, `FrameServiceBase` will guarantee that the mojo
interface object gets deleted as soon as the RFH object gets deleted, therefore,
there is no UAF here. Oh well, no bugs here. Let's move to another mojo
interface implementation... wait... Actually, let's go all the way back to
`BindSmsReceiverReceiver` function.

In particular the line below:

<a id="code-note-7"></a>
```c++
auto* fetcher = SmsFetcher::Get(GetProcess()->GetBrowserContext(), this); // [7]
```

As already mentioned, this function will create (or get an already created)
[SmsFetcher object] and return it [[7]](#code-note-7), let's look further:

<a id="code-note-8"></a>
<a id="code-note-9"></a>
<a id="code-note-10"></a>
```c++
SmsFetcher* SmsFetcher::Get(BrowserContext* context, RenderFrameHost* rfh) {

  auto* stored_fetcher = static_cast<SmsFetcherImpl*>(
      context->GetUserData(kSmsFetcherImplKeyName)); // [8]

  if (!stored_fetcher || !stored_fetcher->CanReceiveSms()) { // [9]
    auto fetcher =
        std::make_unique<SmsFetcherImpl>(context, SmsProvider::Create(rfh));
    context->SetUserData(kSmsFetcherImplKeyName, std::move(fetcher));
  }

  return static_cast<SmsFetcherImpl*>(
      context->GetUserData(kSmsFetcherImplKeyName)); // [10]
}
```

The first thing the code does is check if `BrowserContext` has a `SmsFtecherObject`
stored within it [[8]](#code-note-8), which implies that `SmsFetcher` lifetime is
tied with `BrowserContext` lifetime! If it both exists and can receive SMS message
[[9]](#code-note-9), it will just return a reference to it at
[[10]](#code-note-10).

However, if it cannot receive SMS messages or is not created, it will create a
new `SmsFetcherImpl` object [[9]](#code-note-9). The `SmsFetcherImpl` constructor
expects an [SmsProvider object] that is created by calling its Create method
with our RFH object as an argument. Now, let’s look at the `SmsProvider::Create`
method. (Trivia: Ooops, there was another vulnerability around here:
[1070609]).

<a id="code-note-11"></a>
```c++
// static
std::unique_ptr<SmsProvider> SmsProvider::Create(RenderFrameHost* rfh) {
#if defined(OS_ANDROID)
  if (base::CommandLine::ForCurrentProcess()->GetSwitchValueASCII(
          switches::kWebOtpBackend) ==
      switches::kWebOtpBackendSmsVerification) {
    return std::make_unique<SmsProviderGmsVerification>();
  }
  return std::make_unique<SmsProviderGmsUserConsent>(rfh); // [11]
#else
  return nullptr;
#endif
}
```

There are two `SmsProvider` types:

* `SmsProviderGmsVerification`: Not interesting for us, as it will not take the
  RFH as argument anyway.
* [`SmsProviderGmsUserConsent`][SmsProviderGmsUserConsent]: It'll receive the
  RFH raw pointer as an argument for its constructor [[11]](#code-note-11).
  Looks promising, let's keep looking.

<!-- NOTE: Nicole stopped here -->
<a id="code-note-12"></a>
<a id="code-note-13"></a>
```c++
SmsProviderGmsUserConsent::SmsProviderGmsUserConsent(RenderFrameHost* rfh)
    : SmsProvider(), render_frame_host_(rfh) { // [12]
  // This class is constructed a single time whenever the
  // first web page uses the SMS Retriever API to wait for
  // SMSes.
  JNIEnv* env = AttachCurrentThread();
  j_sms_receiver_.Reset(Java_SmsUserConsentReceiver_create(
      env, reinterpret_cast<intptr_t>(this)));
}


void SmsProviderGmsUserConsent::Retrieve() {
  JNIEnv* env = AttachCurrentThread();

  WebContents* web_contents =
      WebContents::FromRenderFrameHost(render_frame_host_); // [13]
  if (!web_contents || !web_contents->GetTopLevelNativeWindow())
    return;

  Java_SmsUserConsentReceiver_listen(
      env, j_sms_receiver_,
      web_contents->GetTopLevelNativeWindow()->GetJavaObject());
}
```

Oh! So, we will store the raw pointer for the RFH object as a member variable
inside the `SmsProviderGmsUserConsent` [[12]](#code-note-12) class. That looks
dangerous. Also we will end up accessing it whenever we call the Retrieve method
[[13]](#code-note-13). Unless there is some mechanism that ensures the RFH has
not been deleted (spoilers: there aren’t) it may lead to a UAF. Now, to
understand better, let's create an "ownership/reference map", after creating
`SmsFetcherImpl` object, we will end up with something similar to:

<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/SmsFetcherImpl.svg"></center>

As we all love to study chromium code base, one of the things we have learned by
watching [“Anatomy of the browser 201 (Chrome University  2019)"] is that the
`BrowserContext` is pretty much our current `Profile`. This means it'll stay
alive longer than objects like `WebContentsImpl` and `RenderFrameHostImpl`!

We also have learned that we won’t always create a new `SmsFetcherImpl`. Instead,
we will create it once and provide a reference to it every time a new `SmsService`
is created. This smells like a chance for a UAF as we will keep reusing the same
RFH object pointer (inside `SmsProviderGmsUserConsent`) for all new `SmsProvider`
Mojo interface instances.

Indeed, we will have a problem here, as the first time we create a
`SmsProviderGmsUserConsent`, it'll store a reference to the RFH object that
created it. However, we know that `SmsFetcherImpl` will keep reusing
`SmsProviderGmsUserConsent` even after the RFH object is deleted, as there aren’t
any mechanisms to ensure that RFH object hasn’t been deleted!

Therefore, if we have a new RFH object that binds to the `SmsService` interface,
the `SmsService` object will store a raw pointer to the `SmsFetcherImpl` object
containing the `SmsProviderGmsUserConsent` which holds a dangling RFH pointer.

To illustrate it, let’s look at the diagram below.
* Create an iframe and bind to `SmsReceiver`

<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/iframe_bind_1.svg"></center>

* Create another iframe and Bind to `SmsReceiver`

<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/iframe_bind_2.svg"></center>

* Delete the first iframe (aka iframe A), thus, it's iframe A's RFH object gets
  deleted, but we still have a reference to it in `SmsProviderGmsUserConsent`.

* Call receive in `SmsReceiver` for iframe B

<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/iframe_bind_3.png"></center>


As you can see, at the end, iframe B's `SmsService` may end up dereferencing a
freed RFH! Unfortunately, `FrameServiceBase` cannot save us from this problem, in
the end we will have a Use-After-Free issue.

Now, we have found a cool vulnerability, let's try to use it to achieve code
execution in Browser Process context!

[Browser-Interface-Binders]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/browser_interface_binders.cc;drc=8f5b7ee843864f30c9483a8c64afa0433e2e9b90
[Chromium Code Search]: https://source.chromium.org/chromium
[`SmsService::Create`]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/sms/sms_service.cc;drc=28442cacc3be1a7d05a898aba663025a143095ac?originalUrl=https:%2F%2Fcs.chromium.org%2F
[SmsFetcher object]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/sms/sms_fetcher_impl.cc;l=42;drc=28442cacc3be1a7d05a898aba663025a143095ac?originalUrl=https:%2F%2Fcs.chromium.org%2F
[SmsProvider object]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/sms/sms_provider.cc;drc=28442cacc3be1a7d05a898aba663025a143095ac?originalUrl=https:%2F%2Fcs.chromium.org%2F
[1070609]: https://crbug.com/1070609
[SmsProviderGmsUserConsent]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/sms/sms_provider_gms_user_consent.cc;drc=28442cacc3be1a7d05a898aba663025a143095ac;l=22
[“Anatomy of the browser 201 (Chrome University  2019)"]: https://www.youtube.com/watch?v=u7berRU9Qys

# Exploiting the Issue

At this point we know that `SmsProviderGmsUserConsent::Retrieve` will end up using
our freed RFH for some operations. Let's take a look at how exactly it is used:

<a id="code-note-14"></a>
```c++
void SmsProviderGmsUserConsent::Retrieve() {
  JNIEnv* env = AttachCurrentThread();

  WebContents* web_contents =
      WebContents::FromRenderFrameHost(render_frame_host_); // [14]

  if (!web_contents || !web_contents->GetTopLevelNativeWindow())
    return;

  Java_SmsUserConsentReceiver_listen(
      env, j_sms_receiver_,
      web_contents->GetTopLevelNativeWindow()->GetJavaObject());
}


```

First, it'll get a reference to the "freed" RFH and use it as an argument for
the function [`WebContents::FromRenderFrameHost`][from render frame host] [[14]](#code-note-14). Then,
it’ll return a pointer to the `WebContentsImpl` object. Finally, it'll check if
the `WebContentsImpl` isn't nullptr and execute some Java code, otherwise, it'll
just return early.

Now, let's look at the `FromRenderFrameHost` implementation:

<a id="code-note-15"></a>
<a id="code-note-16"></a>
```c++
WebContents* WebContents::FromRenderFrameHost(RenderFrameHost* rfh) {
  if (!rfh)
    return nullptr;

  if (!rfh->IsCurrent() && base::FeatureList::IsEnabled( // [15]
			       kCheckWebContentsAccessFromNonCurrentFrame)) {
    // TODO(crbug.com/1059903): return nullptr here eventually.
    base::debug::DumpWithoutCrashing();
  }

  return static_cast<RenderFrameHostImpl*>(rfh)->delegate()->GetAsWebContents(); // [16]

```

There are two function calls here that use the RFH. The first is at
[[15]](#code-note-15), and the second at [[16]](#code-note-16) where it will
read a member object inside RFH and call its `GetAsWebContents` function. These
method's declarations look like the following:

```c++
virtual bool IsCurrent() = 0;

virtual WebContents* GetAsWebContents();
```

As you can see both methods are declared virtual. As we know, the compiler will
end up creating a [virtual table] to handle the dynamic dispatch! So, if we can
somehow control the "freed" object, and replace its virtual table with a fake
one that we control, we could call an arbitrary function pointer. Once we
can call an arbitrary pointer, we can use Returned-Oriented-Programming (ROP) or
Jump-Oriented-Programming (JOP) and achieve arbitrary code execution.

Also, if we can make the `GetAsWebContents` return nullptr (0x0), we can
smoothly continue the browser execution with no crash. Sounds like a nice plan!

However, we have a problem here: Address Space Layout Randomization (ASLR). We
may have a UAF and we may somehow be able to replace its object virtual table
with controlled contents, but we have no idea where .text, .data or heap
allocations are as we have no information disclosure vulnerability.

We did not get so far to give up! Let's think about it further.

[from render frame host]: https://source.chromium.org/chromium/chromium/src/+/master:content/browser/web_contents/web_contents_impl.cc;l=375;drc=28442cacc3be1a7d05a898aba663025a143095ac
[virtual table]: https://en.wikipedia.org/wiki/Virtual_method_table

### Zygote to Rescue!

I was using a Pixel 3A android device as target for my exploit and while I was
researching a solution for the ASLR problem, I found out that Android has its
own way to launch applications. It is using a concept called "Zygote" and there
are many articles giving in-depth details of how it works and its
[security implications].

For us, Zygote essentially means that every new spawned process will
[share the same ASLR base between them], in another words: Processes can end up
sharing the same virtual memory mapping between some shared libraries!

That is perfect, as having a remote code execution exploit (taking over Renderer
Process by using either a V8 or Blink vulnerability, for example) may help us to
easily defeat ASLR as both Renderer Process and Browser process share the same
virtual mapping between shared libraries.

[security implications]: https://serializethoughts.com/2016/05/25/security-implications-of-zygote-process-creation-model
[share the same ASLR base between them]: https://copperhead.co/blog/aslr-android-zygote/

### Do we really need ROP and/or JOP?
Essentially, once we can replace the "freed" RFH object in memory with
attacker-controlled data, we want to make its virtual table to point to a fake
virtual table and jump into any arbitrary function or a stack-pivot for ROP.
However, ASLR is still a problem for the heap segments as we have no information
about its heap layout.

We can bypass the heap problem by calling another object virtual table that will
end up writing the RFH `this` pointer onto itself (and being able to read the
object memory back into Renderer Process). That should work; however, there is
something even better! [Guang Gong] has presented a nice technique in
["An exploit Chain to Remotely Root Modern Android Devices"]. The article
explains that the `libllvm-glnext.so` (that is present in Pixel 3a) has a
function pointer to `system` in its `.GOT segment`. We can easily replace the RFH
virtual table to point into `libllvm-glnext.so .GOT` and make a call to `system`!


<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/UAF_Exploitation.svg"></center>


The beauty here is that the system's function argument is a pointer to the RFH
object (aka this) that we fully control! Now we can call system with arbitrary
command in context of Browser Process! Feels back into 90s, right?

[Guang Gong]: https://twitter.com/oldfresher
["An exploit Chain to Remotely Root Modern Android Devices"]: https://github.com/secmob/TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices/blob/master/us-20-Gong-TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices-wp.pdf

### Keeping the Browser alive (CRASH != FUN)

Let's look again at the `WebContents::FromenderFrameHost` function but from
another perspective, the ARM-assembly perspective:

<a id="code-note-17"></a>
<a id="code-note-18"></a>
```
0x0000000000000000:  10 B5          push  {r4, lr}
0x0000000000000002:  98 B1          cbz   r0, #0x2c
0x0000000000000004:  04 46          mov   r4, r0

// R0 = RFH->vtable
0x0000000000000006:  00 68          ldr   r0, [r0]
// R1 = RFH->vtable[0xBC/0x4] -- system pointer
0x0000000000000008:  D0 F8 BC 10    ldr.w r1, [r0, #0xbc]
0x000000000000000c:  20 46          mov   r0, r4
// system(R0)
0x000000000000000e:  88 47          blx   r1 // [17]

0x0000000000000010:  30 B9          cbnz  r0, #0x20
0x0000000000000012:  07 48          ldr   r0, [pc, #0x1c]
0x0000000000000014:  78 44          add   r0, pc
0x0000000000000016:  A9 F1 42 EA    blx   #0x1a949c
0x000000000000001a:  08 B1          cbz   r0, #0x20
0x000000000000001c:  A9 F1 06 EE    blx   #0x1a9c2c

// R0 = RFH->member_7c
0x0000000000000020:  E0 6F          ldr   r0, [r4, #0x7c]
// R1 = RFH->member_7c->vtable
0x0000000000000022:  01 68          ldr   r1, [r0]
// R1 = RFH->member_7c->vtable[0x64/0x4]
0x0000000000000024:  49 6E          ldr   r1, [r1, #0x64]
0x0000000000000026:  BD E8 10 40    pop.w {r4, lr} // [18]
// return R1(), where R1 is a function that will set R0 (return value) to 0
// it'll make WebContents == nullptr and not crashing the browser :)
0x000000000000002a:  08 47          bx    r1
0x000000000000002c:  00 20          movs  r0, #0
0x000000000000002e:  10 BD          pop   {r4, pc}
```

As you can see, there are two virtual function calls. The first call,
`RFH->vtable_fptr[0x2F]` [[17]](#code-note-17), we can use to call system with
controlled arguments. However, the second virtual call, `RFH->member_7C->vtable_fptr[0x19]` [[18]](#code-note-18),
is a problem for us. As you already know we have no information disclosure
about the heap memory layout, so, we cannot easily fake a `member_7C` object.

So, what is the solution here? Maybe we could just let the browser crash anyway,
as we will end up executing the system command before the crash happens... But,
let's be honest, crashing the browser isn't fun, can we do something else? Yes,
`Zygote@libllvm-glnext` to the rescue again.

Do we have such a magic pointer in `libllvm-glnext`? Yes! At offset `0x8E4BE8` (`.GOT
segment`) we have exactly what we need, we will end up with the following call
chain:

<center><img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/yarfuaf/UAF_Exploitation_2.svg"></center>


Now, we can both call system and resume execution smoothly without crashing the
browser :)

### Replacing the Object

Alright, at this point we want to replace the object in memory with fully
controlled content. What we need here is some heap-spray primitive. We
could go and try to find our own, however, let's not recreate the wheel. We can
use the same technique demonstrated by
["GPZ Virtually Unlimited Memory"][gpz-virtually-unlimited-memory], since it
still works and fulfills all our needs.

Now, the next step is to find the size of the RFH object size in memory. This is
necessary as we increase the chance to reclaim the memory by spraying payloads
of the same size as the RFH object. You can do it by using your favorite
disassembler, compiler, debugger, or any other tool. In my case, it was `0x880`
bytes.

However, if you heap spray using the technique above, it may work and
reclaim the object, but it may also be a bit unstable. Apparently, on Android,
at least for the version that I had when I wrote the exploit, the Browser
Process will end up using [jemalloc as the default heap allocator].

There is [enough documentation][jemalloc phrack] \(that I recommend reading\)
regarding the allocator internals, thus, I will not go into details here. What
is interesting for us is that jemalloc implements thread specific caches.
Remember, our target object, the freed RFH, is created and destroyed on `UI
thread` and the heap-spray technique will happen on `IO thread`. Due to this, we
may have our allocations happening in different `thread-caches/arenas`.

As we want to be able to reclaim a `freed region` (a.k.a our RFH object) from
another thread, we need to cause either a `flush event` or `hard event` that
will end up freeing some bins/regions inside a `tcache` (each bin has its own
tcache, that is a list for the recently freed regions).

Once a flush or hard event occurs, the region can now be allocated by other
threads. This can be accomplished by first freeing our victim RFH object and
then allocating multiple iframes and freeing them. Following this you can spray
as normal with your heap-spray primitive. In my tests, this seems to have
increased the exploit's reliability.

[jemalloc as the default heap allocator]: https://blog.nsogroup.com/a-tale-of-two-mallocs-on-android-libc-allocators-part-2-jemalloc/
[jemalloc phrack]: http://phrack.org/issues/68/10.html

### Putting It All Together

Now, using all the knowledge we have learned, let's summarize how our exploit
works:

* Create a child iframe that will use `MojoJS` to create and bind to a
  `SmsReceiver` interface (thus, creating a `SmsProviderGmsUserConsent` with a
  pointer to its RFH). `MojoJS` can be enabled by a compromised renderer.
* Send a postMessage from the child iframe to the main frame to tell it the
  Mojo interface has been created. The main frame can now delete the child
  iframe with `document.body.removeChild`.
* In the main frame, create another `SmsReceiver` interface. This instantiation
  will use the already created `SmsFetcherImpl` which has a raw pointer to the
  `freed` RFH object.
* Prepare our heap payload:
  * The first 4 bytes (32 bit architecture) is the virtual table pointer, it'll
    be a pointer to `libllvm-glnext.so` `.got.plt` minus `0xBC` (offset for
    virtual table) so we land in the correct address.
  * The next bytes will be our shell command, something of the form "||
    (command)." This way it'll first execute the virtual table address as a
    "command" and then execute our shell command. For the exploit, I used:
    `' || (toybox nc -p 4444 -l /bin/sh)')`.
  * At offset `0x7C` of our payload we will have a pointer to the "magic
    function pointer" in `libllvm-glnext.so`, so we can guarantee the
    `GetAsWebContents` virtual method will return the value `0x0`, making
    `SmsProviderGmsUserConsent::Retrieve` return early, avoiding a browser
    crash.
  * Spray these bytes using the same technique learned
    [here][gpz-virtually-unlimited-memory], but use the jemalloc trick described
    earlier to make it more reliable.
* Call `SmsReceiver.receive` method and watch the magic!

You can find the final exploit <a href="https://gist.github.com/ohnull/2cbfa501936a2fff4fd9efa67310cda8">here</a>.

[gpz-virtually-unlimited-memory]: https://googleprojectzero.blogspot.com/2019/04/virtually-unlimited-memory-escaping.html

# Conclusion

At this point you can run shell commands in the context of Browser Process. Due
to the Android security model, you may have limited resource access as you are
still inside Android's application sandbox. The next step would be to chain a
kernel vulnerability, as described [here][gong-tiyunzong], but this is a story
for another day.

I hope you have enjoyed reading and learning a little more about Chromium as
much as I have while learning and writing all of it. This issue was a
nice exploit exercise and I think it would have been harder to exploit if Zygote
didn't weaken ASLR on Android. Now that we know how the vulnerability works and
its pattern, we can write more security documentation, give insight to our
developers into how to write Mojo interfaces with no such pattern and proactively
find vulnerabilities on our security reviews.

Furthermore, Google has been working hard to mitigate UAF issues through efforts
such as [PartitionAlloc everywhere], [MiraclePtr] and [*Scan].
We are looking forward to making contributions and working with them to make
these vulnerabilities harder to exploit.

[gong-tiyunzong]: https://github.com/secmob/TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices/blob/master/us-20-Gong-TiYunZong-An-Exploit-Chain-to-Remotely-Root-Modern-Android-Devices-wp.pdf
[PartitionAlloc everywhere]: https://bugs.chromium.org/p/chromium/issues/detail?id=1121427
[MiraclePtr]: https://www.youtube.com/watch?v=ohlxw5kDn-k
[*Scan]: https://source.chromium.org/chromium/chromium/src/+/master:base/allocator/partition_allocator/pcscan.h