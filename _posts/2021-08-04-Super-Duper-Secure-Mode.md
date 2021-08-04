---
title: Super Duper Secure Mode
author: Johnathan Norman
date: 2021-08-04 8:50:00 -7000
categories: [Vulnerabilities]
tags: [JIT]
math: true
author_twitter: spoofyroot
---

# Introduction

The VR team is experimenting with a new feature that challenges some
conventional assumptions held by many in the browser community. Our hope
is to build something that changes the modern exploit landscape and
significantly raises the cost of exploitation for attackers. Mitigations
have a long history of being bypassed, so we are seeking feedback from
the community to build something of lasting value.

Most importantly we plan to have fun with this project. This includes
giving the experiment a slightly provocative name because we think it is
funny, and it is a bit too early for something official.

# Security vs. Performance: Reconsidering the trade-offs 

We bounce between offensive and defensive roles on this team quite
often. When working on browser exploits, our favorite target is always
V8. JavaScript engine bugs are a mainstay for attackers for a variety of
reasons; they provide powerful exploit primitives, there is a steady
stream of bugs, and exploitation of these bugs often follows a
straightforward template. JavaScript engine exploitation has [not changed much over the years](http://phrack.org/papers/attacking_javascript_engines.html) and
generally follows the same pattern:

-   Fake an object
-   Get AddrOf Primitive
-   Achieve arbitrary write

We can effectively copy/paste our bug into a template and have something
working fairly quickly. Attackers even have frameworks like
[PwnJS](https://github.com/theori-io/pwnjs) which allow for this quick
conversion.

However, as a defender, this is a nightmare. The regular stream of bugs
requires frequent security updates, and the ease of exploitation means
attackers can quickly weaponize exploits, which is helpful when abusing
the [patch gap](https://blog.exodusintel.com/2019/09/09/patch-gapping-chrome/).
This problem is not unique to V8, this is a common problem among most
modern JavaScript engines. Google, Mozilla, Microsoft, and others try to
mitigate this risk proactively with large investments in static
analysis, bug bounties, and fuzzing. All of these allow for rapid
identification of some of these issues, but inevitably, a number are
missed. JavaScript engines remain a remarkably difficult security
challenge for browsers.

Why? Well, the problem is in part due to a performance technology called
"Just-In-Time Compilation" (JIT). JITs were introduced to browsers in
[2008](https://en.wikipedia.org/wiki/SpiderMonkey#TraceMonkey) to speed up specific tasks in JavaScript. JIT-enabled engines
effectively take loosely-typed JavaScript and compile it to machine code
just before it is needed. This process is sometimes referred to as
"speculative optimization." JavaScript code is optimized through a
series of complex processing pipelines. These changes result in
performance gains that are quite impressive. Developers have managed to
make JavaScript performance [comparable with C++](https://www.linkedin.com/pulse/algorithmic-performance-comparison-c-javascript-java-malyshev), which is an impressive
feat. However, a lot goes into this process. To provide some
perspective, here is a high-level overview from a [V8 presentation](https://docs.google.com/presentation/d/1H1lLsbclvzyOF3IUR05ZUaZcqDxo7_-8f4yJoxdMooU/edit#slide=id.p)
produced by Google in 2016.

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/turbofan.png" 
alt="TruboFan processing pipeline chart"></p>



<sup>1</sup><sub>[https://docs.google.com/presentation/d/1H1lLsbclvzyOF3IUR05ZUaZcqDxo7\_-8f4yJoxdMooU/edit#slide=id.p](https://docs.google.com/presentation/d/1H1lLsbclvzyOF3IUR05ZUaZcqDxo7\_-8f4yJoxdMooU/edit#slide=id.p)</sub>

The above chart is just one phase of the entire V8 processing pipeline.
This does not include the parsers, interpreter, the recently-added
second JIT called [Sparkplug](https://v8.dev/blog/sparkplug), or many other components. This is a
remarkably complex process that very few people understand and it has a
small margin for error.

Performance and complexity often come at a cost, and often we bear this
cost in the form of security bugs and subsequent patches. Looking at CVE
(Common Vulnerabilities and Exposures) data after 2019 shows that
roughly 45% of CVEs issued for V8 were related to the JIT engine.
Moreover, we know that attackers weaponize and abuse these bugs as well;
[an analysis from
Mozilla](https://docs.google.com/spreadsheets/d/1FslzTx4b7sKZK4BR-DpO45JZNB1QZF9wuijK3OxBwr0/edit#gid=0)
shows that over half of the "in the wild" Chrome exploits abused a JIT
bug, as illustrated in the charts below. Note "Edge" below refers to the
legacy version of Edge.

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/vuln_by_type.png" 
alt="Vulnerabities by type chart"></p>

<sup>2</sup><sub>
[https://docs.google.com/spreadsheets/d/1FslzTx4b7sKZK4BR-DpO45JZNB1QZF9wuijK3OxBwr0/edit#gid=865365202](https://docs.google.com/spreadsheets/d/1FslzTx4b7sKZK4BR-DpO45JZNB1QZF9wuijK3OxBwr0/edit#gid=865365202) </sub>

# Is JIT worth it? 

Traditionally, browser developers are willing to accept this cost
because users want their browsers to be "fast" but what if we simply
disabled the JIT? This reduction of attack surface has potential to
significantly improve user security; **it would remove roughly half of
the V8 bugs that must be fixed**. For users, this means less frequent
security updates and fewer emergency patches required. These updates and
patches are common points of frustration for our customers, particularly
those in large enterprise environments who must test updates before
rolling them out.

There are benefits beyond just attack surface reduction. Due to how the
V8 JIT works, several impactful mitigation technologies cannot be
brought to bear in the renderer process. For example,
[Controlflow-Enforcement Technology](https://software.intel.com/content/www/us/en/develop/articles/technical-look-control-flow-enforcement-technology.html)
(CET), a new hardware-based exploit mitigation from Intel, was disabled.
Similarly, Arbitrary Code Guard (ACG) was not enabled due to the use of
RWX memory pages in the process. This is unfortunate because the
renderer process handles untrusted content and should be locked down as
much as possible. By disabling JIT, we can enable both mitigations and
make exploitation of security bugs in any renderer process component
more difficult. Microsoft's [Matt Miller laid out this strategy in early
2017](https://blogs.windows.com/msedgedev/2017/02/23/mitigating-arbitrary-native-code-execution/).

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/strategy.png" 
alt="Mitigation Strategy"></p>

This reduction in attack surface kills half of the bugs we see in
exploits and every remaining bug becomes more difficult to exploit. To
put it another way, we lower costs for users but increase costs for
attackers.

# But what about performance? 

Performance is important to the Edge team, and we are quite proud of the
[improvements we have
made](https://blogs.windows.com/msedgedev/2021/03/04/edge-89-performance/).
It is a complicated topic that includes a wide range of issues including
startup time, memory consumption, rendering time and more. JavaScript
plays a key role in any browser story. JITs exist for a reason and that
is to optimize JavaScript performance.

We looked at the results of disabling the JIT in our lab. The Edge
performance lab runs hundreds of tests that mimic real-world sites and
conditions to look at the impact of a change. We group these tests into
categories such as power, startup, memory, and page load. First, let's
look at how many tests saw an improvement, regression, or no change.

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/number_of_tests.png" 
alt="Number of Tests Chart"></p>

We see that most tests see no changes with JIT disabled. There are a few
improvements and regressions, but most tests remain unchanged.
**Anecdotally, we find** **that users with JIT disabled rarely notice a
difference in their daily browsing.** But how much variation did we see
in the tests that changed? The chart below shows the average percentage
improvement or regression in performance.

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/improvement_and_regression.png" 
alt="Average Improvement and Regression Chart"></p>

We find that disabling the JIT does not always have negative impacts.
Our tests that measured improvements in power showed 15% improvement on
average and our regressions showed around 11% increase in power
consumption. Memory is also a mixed story with negatively impacted tests
showing a 2.3% regression, but a larger gain on the tests that showed
improvements. Page Load times show the most severe decrease with tests
that show regressions averaging around 17%. Startup times, however, have
only a positive impact and no regressions.

It is important to note that these results do not include Speedometer
2.0, the most referenced benchmark. Disabling JIT does result in
significantly lower scores in JavaScript benchmarks. Our tests showed a
decline as high as 58%. However, often users do not notice this impact
because this benchmark tells only part of a larger performance story.

So, are the performance gains provided by JIT worth the resulting
security bugs, updates, and foregone security mitigations? The answer to
that question is dependent on several factors. Are you viewing a blog or
playing a game online? Are you visiting a trusted or untrusted site? As
we continue our journey with this experiment, we will look to explore
these factors and the impacts they can have on our users.

# Project Super Duper Secure Mode 

Over the next few months, we will try to answer these questions with our
Super Duper Secure Mode (SDSM) experiment. It will take some time, but
we hope to have CET, ACG, and CFG protection in the renderer process.
Once that is complete, we hope to find a way to enable these mitigations
intelligently based on risk and empower users to balance the tradeoffs.

Currently, SDSM disables JIT (TurboFan/Sparkplug) and enables CET. At
the moment, Web Assembly is not supported in this mode. We hope to
slowly enable new mitigations and add Web Assembly support over the next
few months as we continue testing and experimentation. You can find the
feature under edge://flags in Edge Canary, Dev, and Beta. 

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/sdsm/sdsm.png" 
alt="Super Duper Secure Mode Flag"></p>

This is of course just an experiment; things are subject to change, and
we have quite a few technical challenges to overcome. Also, our
tongue-in-cheek name will likely need to change to something more
professional when we launch as a feature. For now, we are going to
continue having fun with it.

If you decide to test the feature, please send us your feedback through
the Feedback menu in Microsoft Edge. We are eager to hear about your
experience.
