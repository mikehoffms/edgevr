---
title: A Story of a Bug Found Fuzzing
author: Abdulrahman Alqabandi
date: 2022-06-10 8:30:00 -0700
categories: [Vulnerabilities]
tags: [fuzzing, tips]
math: true
author_twitter: qab
---
In a previous blogpost it covered and mentioned automation and how it is great
at finding memory issues. We also got some feedback to expand on fuzzing, so
this post will cover how we came to develop a fuzzer and how it found its first
security issue early in development.

The main intention of this fuzzer is to use the signal from MSRC cases and see
if it can find the next bug before it gets reported which follows the same
pattern. The result was a cool browser fuzzer and the experiment yielded
interesting results.

# The Target
We noticed a pattern in recent memory corruption bugs affecting both Edge and
Chromium where an extension was used as a proof of concept. This was
particularly interesting to me because I
[looked at extensions](https://leucosite.com/WebExtension-Security-Part-2/) a
few years ago and only found logic bugs and, with an itch to make an
experimental fuzzer why not try to create an extension based fuzzer for some
variant hunting.

Now that I have a general component (Web Extensions) as a target, where to
start?

When reading through all of the publicly disclosed chromium bugs that involved
an extension and a browser crash, two bugs from
[David Erceg](https://twitter.com/david_erceg) stood out
([1188889](https://bugs.chromium.org/p/chromium/issues/detail?id=1188889),
[1190550](https://bugs.chromium.org/p/chromium/issues/detail?id=1190550)) where
the `chrome.debugger.sendCommand` was used and it was interesting.

The `chrome.debugger` extension API allows you to control some tabs using the
[devtools protocol](https://chromedevtools.github.io/devtools-protocol/), this
is the same protocol remote debugging uses. The function `sendCommand` stood out
which looks like the following:

```javascript
chrome.debugger.sendCommand(
  target: Debuggee,
  method: string,
  commandParams?: object,
  callback?: function,
)
```
This looks like a promising function to start fuzzing.

# Gathering More Information
Now that a more specific API was identified, let’s look at what kind of input it
takes:

1.	**target**: This is a simple object that contains either the extensionId,
   tabId or targetId of the target.
2.	**method**: This is a string that must contain valid values, can’t generate
   random strings here.
3.	**commandParams**:
   “[JSON object with request parameters. This object must conform to the remote debugging params scheme for given method.](https://developer.chrome.com/docs/extensions/reference/debugger/#method-sendCommand)”
   Also can’t generate random strings here for param names.
4.	**callback**: expected

In order to create valid and relevant method and command parameters, we need to
look up and learn about this devtools protocol. Thankfully, the devtools folks
have a
[JSON file describing all these method names](https://github.com/ChromeDevTools/devtools-protocol/tree/master/json)
and their parameters including what type of values they expect. Perfect!

```json
{
    "version": {
        "major": "1",
        "minor": "3"
    },
    "domains": [
        {
            "domain": "Accessibility",
            "experimental": true,
            "dependencies": [
                "DOM"
            ],
            "types": [
                {
                    "id": "AXNodeId",
                    "description": "Unique accessibility node identifier.",
                    "type": "string"
                },
                {
                    "id": "AXValueType",
                    "description": "Enum of possible property types.",
                    "type": "string",
                    "enum": [
                        "boolean",
                        "tristate",
                        "booleanOrUndefined",
                        "idref",
                        "idrefList",
                        "integer",
                        ..
                        ...
```
This means the extension that eventually gets installed on the browser can
consume this JSON file and use its data to construct somewhat valid devtools
protocol commands. Since the types are described, it can generate the value of
any given type with random values conveniently.

But wait, if this fuzzer needs to install an extension on the fly, how can that
be achieved?

# The Harness

The great thing about Chromium is that there are a lot of run time flags that do
all sorts of things. In this case there are a couple of flags that will load
extensions on startup:

`--disable-extensions-except="C:/ext/"` 

`--load-extension="C:/ext/"`

(You can find a list of these flags
[here](https://peter.sh/experiments/chromium-command-line-switches/), though not
actively maintained so make sure to verify by
[looking up the flag in chromium source](https://source.chromium.org/search?q=%22--no-sandbox%22))

This harness will be simple and straightforward, all it needs is to control the
launching of the target binary (the browser) with command line flags as well as
handle/detect all types of crashes. Additionally, it needs to log everything and
make sure the fuzzing process runs continuously. Sometimes it’s the job of the
harness to feed testcases into the browser but for this case we only load one
‘testcase’ (an extension) that continuously runs until a crash or timeout then
the harness repeats.

This basic harness is using NodeJS and it does the following:

1.	Contain a list of useful flags (like `--no-sandbox` which is important
   in ASan builds as otherwise we won’t see child process crashes)
2.	[Spawn](https://nodejs.org/api/child_process.html#child_processspawncommand-args-options) browser using the flags
    - Save all [stdout](https://en.wikipedia.org/wiki/Standard_streams) from the
      browser process
3.	Handle gracefully when browser crashes
    - Check [crash signal](https://nodejs.org/api/child_process.html#event-exit)
    - Save ASan report and `stdout` logs
4.	Clean up and restart

# The Extension

As mentioned before, the way it will be fuzzing the extension API is by loading
one giant extension and listen for crashes. It seems much faster than creating
thousands of extensions and loading them, but with this speed come some
drawbacks.

The idea of this extension is for it to have all the permissions that are
relevant to us, in this case the `debugging` permission and others which can be
helpful (`tabs` and `<all_urls>` host permission). Once it loads it will consume
the devtools protocol JSON ‘database’ and using the data there it creates
somewhat valid devtools protocol commands.

Some sample generated commands:

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/fuzzer_bug/extuiconsole.gif" alt="console output showing devtool commands"/>

We’re not paying much attention to the semantics of the fuzzed values as far as
parameter values (the names should be valid), some will throw errors and that’s
fine for now. We want to quickly get our fuzzer running and then improvements
can be made.

As soon as all the pieces were connected and the fuzzer was running, a lot of
crashes started occurring and other annoyances like randomly generated commands
can result in either the extension crashing or stalling. Once these initial
“fuzz blocker” bugs got fixed the fuzzer ran smoothly for a short time before
finding the first valid security bug.

```
ERROR: AddressSanitizer: heap-buffer-overflow on address 0x11f305796100 at pc 0x7ff8dd649dfc bp 0x00c8065fd2f0 sp 0x00c8065fd338
READ of size 2 at 0x11f305796100 thread T0
    #0 0x7ff8dd649dfb in std::__1::basic_string<char16_t,std::__1::char_traits<char16_t>,std::__1::allocator<char16_t> >::basic_string<std::nullptr_t> C:\b\s\w\ir\cache\builder\src\buildtools\third_party\libc++\trunk\include\string:835
    #1 0x7ff8dd649a84 in mojo::StructPtr<blink::mojom::KeyData>::StructPtr<const int &,const int &,const int &,const int &,const bool &,const bool &,const char16_t (&)[4],const char16_t (&)[4]> C:\b\s\w\ir\cache\builder\src\mojo\public\cpp\bindings\struct_ptr.h:63 
```

This is always a great sign, especially early in development. However, some of
the early decisions came back to bite. The decision to only have one extension
rather than generating thousands of extensions meant we do not have a
conventional testcase that reproduced this crash once a crash occurred. Instead,
we rely on logging to see which commands were fired before the crash. The logs
showed the following command call right before crash:

```javascript
      chrome.debugger.sendCommand({ tabId: atabid }, "Input.dispatchKeyEvent", {
        type: unescape("keyDown"),
        modifiers: -1,
        TimeSinceEpoch: -129,
        text: unescape("%uA8E9%5E%uD866%uDD5E"),
        unmodifiedText: unescape("%uD834%uDD6F%uDB43%uDD46"),
        keyIdentifier: unescape("%26%3D%uDB40%uDCF6%u0660"),
        code: unescape("%7C%uE468"),
        key: unescape("%uC79B%uD84A%uDFF9%uD834%uDD66%uDB42%uDDE5%uD85E%uDFD1"),
        windowsVirtualKeyCode: -128,
        nativeVirtualKeyCode: 16,
        autoRepeat: false,
        isKeypad: false,
        isSystemKey: false,
        location: 4096,
        commands: ["≤áÄí-"],
      });
```

When running this command manually in a plain extension it did not crash the
browser. After some digging around it turns out we had to run it multiple times
as we opened a new window as a debug target. The reliably reproducible extension
testcase looked like this now:

```javascript
var atabid;
chrome.windows.create({ url: "about:blank", type: "normal" }, (w) => {
  atabid = w.tabs[0].id;
  chrome.debugger.attach({ tabId: atabid }, "1.3", function () {
    setInterval((g) => {
      chrome.debugger.sendCommand({ tabId: atabid }, "Input.dispatchKeyEvent", {
        type: unescape("keyDown"),
        modifiers: -1,
        TimeSinceEpoch: -129,
        text: unescape("%uA8E9%5E%uD866%uDD5E"),
        unmodifiedText: unescape("%uD834%uDD6F%uDB43%uDD46"),
        keyIdentifier: unescape("%26%3D%uDB40%uDCF6%u0660"),
        code: unescape("%7C%uE468"),
        key: unescape("%uC79B%uD84A%uDFF9%uD834%uDD66%uDB42%uDDE5%uD85E%uDFD1"),
        windowsVirtualKeyCode: -128,
        nativeVirtualKeyCode: 16,
        autoRepeat: false,
        isKeypad: false,
        isSystemKey: false,
        location: 4096,
        commands: ["≤áÄí-"],
      });
    }, 100);
  });
});
```

Once we were able to reliably crash the browser it turned out to be an upstream
issue and so it was reported to them. The bug report is now public and you can
see it [here](https://bugs.chromium.org/p/chromium/issues/detail?id=1276331).

# What Happened
The bug is due to `unmodifiedText` (line 11 in the last screenshot) was handled
in the browser process without taking account of null termination which lead to
a
[heap buffer overflow](https://learn.microsoft.com/cpp/sanitizers/error-heap-buffer-overflow?view=msvc-170#example---classic-heap-buffer-overflow)
(read), as [caseq@](https://bugs.chromium.org/u/2585305731/) (assigned to this
bug) mentions: “…That said, this may lead to us sending a piece of browser
memory into renderer.”

Let’s take a closer look:

```cpp
#include <uchar.h>
#include <stdio.h>
#include <string>

int main(void)
{
    char16_t char16[] = u"ABCD";
    std::u16string text16(u"ABCD");
    printf("size of char16 = %zu\n", sizeof char16 / sizeof *char16);
    printf("size of text16 = %zu\n", text16.size());
    
}
```
Out:
```
PS C:\Users\test\Desktop\mkhtbr> ./test.exe
size of char16 = 5
size of text16 = 4
```

Note the above output; `char16_t` should end with a null byte which is why its
size is bigger than the `u16string`. Let’s modify this to replicate the same
underlying bug found:

```cpp
#include <uchar.h>
#include <stdio.h>
#include <string>

int main(void)
{
    char16_t *char16=(char16_t*)malloc(4 * sizeof(char16_t));
    std::u16string text16(u"ABCD");

    printf("size of char16 = %zu\n", sizeof char16 / sizeof *char16);
    printf("size of text16 = %zu\n", text16.size());

    for (size_t i = 0; i < text16.size(); ++i){
        char16[i] = text16[i];
    }

    // Try to access everything in the 'char16_t' array assuming '4' is limit.
    for (size_t n = 0; n <= 4; n++) {
        printf("n=%#x & ", n);
        printf("char16[n]=%#x\n", char16[n]);
    }
}
```

We compile using AddressSanitizer by executing the following:
```
clang++ -O1 -g -fsanitize=address -fno-omit-frame-pointer -o a.exe .\old.cpp 
```
Finally, we run it:

```
PS C:\Users\test\Desktop\mkhtbr> ./test.exe
size of char16 = 4
size of text16 = 4
n=0 & char16[n]=0x41
n=0x1 & char16[n]=0x42
n=0x2 & char16[n]=0x43
n=0x3 & char16[n]=0x44
n=0x4 & =================================================================
ERROR: AddressSanitizer: heap-buffer-overflow on address 0x11cdab3a0038 at pc 0x7ff780c517d1 bp 0x0044ccf0f4e0 sp 0x0044ccf0f528
READ of size 2 at 0x11cdab3a0038 thread T0
    #0 0x7ff780c517d0 in main C:/Users/test/Desktop/mkhtbr/./test.cpp:20:35
```

As you can see, this is the same crash as the Chromium bug. Upstream fixed it
swiftly and correctly guessed it was a bug that’s been there for a while (which
meant it was likely in the stable version affecting all users).

# Conclusion
We’ve only covered the initial development of this fuzzer, we’re continuously
developing it by adding more APIs and running it during development. So far it
has found the following security issues:

<table>
<thead>
  <tr>
    <th>ID</th>
    <th>Title</th>
    <th>Upstream</th>
    <th>Severity</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>N/A</td>
    <td>Heap UaF in edge::fre:: █ █ █ █:: █ █ █ █ █ </td>
    <td>No</td>
    <td>High</td>
  </tr>
  <tr>
    <td><a href="https://bugs.chromium.org/p/chromium/issues/detail?id=1297404">
    1297404</a></td>
    <td>Heap UaF in global_media_controls::MediaItemManagerImpl::HideItem</td>
    <td>Yes</td>
    <td>High</td>
  </tr>
  <tr>
    <td><a href="https://bugs.chromium.org/p/chromium/issues/detail?id=1283077">
    1283077</a></td>
    <td>Heap buffer overflow read in webui tabstrip</td>
    <td>Yes</td>
    <td>High</td>
  </tr>
 <tr>
    <td><a href="https://bugs.chromium.org/p/chromium/issues/detail?id=1276331">
    1276331</a></td>
    <td>Heap buffer overflow read in blink::mojom::WidgetInputHandlerProxy::DispatchEvent</td>
    <td>Yes</td>
    <td>High</td>
  </tr>
</tbody>
</table>

Here is what it looks like currently when running it, it gets progressively
chaotic as time goes by (not sped up and using ASan).

<img src="{{ site.url }}{{ site.baseurl }}/assets/img/blog_img/fuzzer_bug/extui.gif" alt="screen recording of the fuzzer opening several UI"/>

# Coming Soon

We will soon be posting another guest blogpost by
[David Erceg](https://twitter.com/david_erceg) where he will discuss some of the
bugs he found as well as sharing his methodology. Also included in the post will
be some of the ways the Edge Security Team reacted to his reports to find
variants.

Until then,
[check out his first guest blogpost](https://microsoftedge.github.io/edgevr/posts/attacking-the-devtools/)
if you haven't already.