---
title: Introducing Enhanced Security for Microsoft Edge
author: Johnathan Norman
date: 2022-02-16 8:00:00 -0700
categories: [Features]
tags: [JIT]
math: true
author_twitter: spoofyroot
---

# Introduction

Today, we’re excited to announce some improvements to our experimental security
feature in Microsoft Edge 98. This is a continuation of our previous
[post][old-post]. If you have not read it, we would suggest checking it out
before jumping into this post.

While we’re still experimenting with this feature, we can share the initial
results and user experience are promising. Most users who have turned on the
feature have not noticed any issues or reported any tradeoffs, including some of
our peers who were unaware we had enabled it for them in the Microsoft Edge
canary channel.

[old-post]: https://microsoftedge.github.io/edgevr/posts/Super-Duper-Secure-Mode/

# More Protections and More Flexibility

## Additional Protections

Microsoft Edge already takes advantage of advanced protections like
[Code Integrity Guard (CIG)][cig] and [Control Flow Guard (CFG)][cfg]. As of
Microsoft Edge 98, [Control-flow Enforcement Technology (CET)][cet] and
[Arbitrary Code Guard (ACG)][acg] will be enabled in the renderer process when a
site is in enhanced security mode. These additional mitigations prevent dynamic
code generation in the renderer processes and implement a separate shadow stack
to protect return addresses. Moreover, we are quite excited that Microsoft Edge
now supports both forwards and backwards control-flow protection. By applying
these protections, we can provide defense in depth that spans beyond JIT attacks.

[cig]: https://learn.microsoft.com/microsoft-365/security/defender-endpoint/exploit-protection-reference?view=o365-worldwide#code-integrity-guard
[cfg]: https://learn.microsoft.com/windows/win32/secbp/control-flow-guard
[cet]: https://www.intel.com/content/www/us/en/developer/articles/technical/technical-look-control-flow-enforcement-technology.html
[acg]: https://learn.microsoft.com/microsoft-365/security/defender-endpoint/exploit-protection-reference?view=o365-worldwide#arbitrary-code-guard

## Manual and Automatic Site bypass lists

Our goal is to protect as many people as possible while also making sure most
end users don’t notice an impact to their normal tasks. One way to balance user
security and reduce impact is to bypass security protections for frequently
visited sites. 

Right now, we’re experimenting with how to deliver unique, user-tailored bypass
lists. Unique lists could help reduce the chance that an XSS vulnerability in
any particular site be used to deliver an exploit. Bypass lists that are unique
to each user mean an attacker would have to predict which sites a user trusts,
which makes exploitation more difficult.

To curate this list and make sure it represents user actions, our experiment
builds on top of Chromium project’s site engagement score as a measure of how
engaged a specific user is with a specific site. Site engagement scores can
range from 0 (meaning a user has no relationship with a site) to 100 (meaning
that a user is extremely engaged with a site). Activities such as browsing to a
site repeatedly/over several days, spending time interacting with a site, and
playing media on a site all cause site engagement scores to increase, whereas
not visiting a site causes site engagement scores to decay exponentially over
time. You can view your own site engagement scores by navigating to
edge://site-engagement in Microsoft Edge.

In this experiment, new users to Microsoft Edge, who use balanced mode, and do
not have any engagement score data will utilize a list of top-traffic sites that
will decay off as the user browses.

As the name “Balanced Mode” suggests, we will continue to experiment to make
sure we can maximize user protection with minimum impact to a user’s normal
tasks.

You can learn more about the exception site list and how Microsoft Edge supports
enhancing your security on the web through its options to control this feature
in the [documentation to Browse more safely with Microsoft
Edge][documentation].

[documentation]: https://learn.microsoft.com/DeployEdge/microsoft-edge-security-browse-safer

## Linux and Mac Support

In addition to these new user controls, we have added support for both Mac
(Intel/M1) and Linux users. Starting with Microsoft Edge version 99.0.1140.0 on
Linux, we added an experimental flag to emulate the ACG feature used on the
Windows platform. This can be enabled from edge://flags. We hope to add more
platform-specific mitigations for these platforms in the future.

<p align="center"><img src="{{ site.baseurl }}/assets/img/blog_img/safe/experiments.png" 
alt="ACG feature in experiments page"></p>

## Introducing Drum Brake, a WebAssembly Interpreter

A key challenge in enabling ACG in the renderer process is supporting
WebAssembly (WASM). V8 makes use of a compiler to convert WASM code into machine
instructions and this requires writable and executable pages in memory. This
memory allocated for WASM is often used by attackers to execute their own code
in exploits. Enabling ACG prevents this, but it also breaks WASM. To address the
lack of WASM support we decided to build a WASM interpreter which would allow us
to enable ACG.

To enable this, we’re building a new WASM Interpreter code named
“DrumBrake.” It seeks to provide a secure WASM environment that unblocks the most
common WASM use cases without requiring JIT. We expect to see some
[tradeoffs][drumbrake-perf] for sites running WASM in DrumBrake. For example,
Drum Brake does require less memory, which is a nice bonus but we expect that
compute-intensive applications may not perform as well. 

We plan to contribute DrumBrake to the Chromium Project for others to use as
well. You can find the software spec [here][drumbrake-doc] if you are interested
in how the code works. Thanks to the various members of the V8 team who gave us
feedback on the design!

Lastly, we are on track to landing the WASM interpreter starting with the canary
Edge 101 channel later this month. This will be followed with some
experimentation, further testing, and stabilization before rolling out under the
Stable channel.

[drumbrake-perf]: https://docs.google.com/document/d/1OIJ4Sv2XfTlI5NmTS1QI8v8wPL0LUT5s1W2D9OlJmMc/preview#heading=h.8enk175z61od
[drumbrake-doc]: https://docs.google.com/document/d/1OIJ4Sv2XfTlI5NmTS1QI8v8wPL0LUT5s1W2D9OlJmMc/preview#heading=h.2hff7nffvheq
# What’s Next? 

One of the most revealing aspects of this work was that 84% of users who opted
into the feature were satisfied with tradeoffs. Suggesting such an idea among
our peers 6 months ago would have resulted in us being laughed at out of the
room. It is worth mentioning that when we originally had this idea, we doubted
our Microsoft Edge peers would even consider it. We quietly made changes to our
browser without explicitly telling them the specifics and then asked them weeks
later to see if they noticed the change. They would always say no, and only then
would we inform them that we disabled the JIT. After surprising multiple
developers in Microsoft Edge, we got the support needed to try this experiment.
One can’t help but wonder what other well established assumptions about users
and the web we should reconsider.

IT Admins and Enterprise users who have experienced enhanced security mode with
Microsoft Edge shared their positive experiences with statements like, “Didn’t
see any problems when using it”, “I noticed zero difference despite the concern
of a JITless browser on performance”, and “No noticeable changes since enabling
last week”. As we mentioned above, our sample size needs to increase, and we
also need to gather feedback regarding the new WASM experience once DrumBrake
reaches the canary channel later this month. We think it is safe to say there is
user value for this feature and further experimentation is warranted.

The VR team is a specialized team focused on bug finding, addressing threats and
exploitation. We don’t normally develop features. We ‘break’ things. And as
such, this has been a fun new challenge as we received a glimpse of walking a
mile in the shoes of ‘makers’. As the feature grows, we expect to continue
supporting the feature’s progress and promise to build customer confidence when
browsing more safely with Microsoft Edge. The team has appreciated and learned
the amount of care and detail it takes to ship and experiment with features,
which provided new insights and perspectives into our future approach for
breaking new things.

# Send us feedback

We want to get your feedback on our next iteration to enhance your security on
the web. If something looks broken, or if you have feedback to share on these
changes, we want to hear from you. You can reach out to Microsoft Support to
report issues or feedback. You can also leave feedback in our [TechCommunity
forum][tech-forum] or Send feedback **(Alt+Shift+I)** inside Microsoft Edge.

[tech-forum]: https://techcommunity.microsoft.com/t5/enterprise/bd-p/EdgeInsiderEnterprise