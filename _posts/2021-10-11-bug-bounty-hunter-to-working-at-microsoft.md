---
title: Bug bounty hunter to working at Microsoft
author: Abdulrahman Alqabandi
date: 2021-10-11 8:30:00 -0700
categories: [Perspective]
tags: [tips]
math: true
author_twitter: qab
---

I usually write about achievements in the form of a browser bug that I found
interesting, in hopes that someone reading will find it useful in their own bug
hunting pursuits. However, in this blog post I will be going into the
differences between bug hunting as a hobby and vulnerability research as a job.
I will go through my regrets, surprises, and other advice.

This sort of transition gave me a unique perspective which I would love to
share. Please keep in mind that this blog post is completely my own subjective
experience and most likely will not reflect your own. But I hope I leave you
with some insight that I wish I had at the time.

# Background

I’ve been active in bug bounty hunting since the start of 2015 with my first
official bounty received
[in early](https://twitter.com/Qab/status/586627244489109504) 2015. This first
bug was an XSS in O365 Outlook and by late 2015 I had found my first valid
browser [bug in Firefox](https://twitter.com/Qab/status/660118656618135553).
Ever since then I pretty much stuck to finding bugs in browsers.

During the last 5 years I went from a university dropout, to an assistant
systems developer, then a network security engineer and finally today I find
myself working at Microsoft with the Browser Vulnerability Research team.
Something that seemed like a dream turned to reality.

# Hindsight is 20/20

A big mistake during my bug bounty hunting was the false assumption that me
focusing on browser bugs (almost) exclusively meant that my flavor of
information security cannot translate to any career under the umbrella of
security. This false assumption was further fed by the fact that browser
vulnerability research is a somewhat niche field, it looked like most bug
hunters I knew early on started off with web application security and usually
stick to it whilst I became fascinated with browsers and dove into it head on.

The reality is that because browser security is such a niche meant that I am
part of a short list of candidates eligible for a browser security position at
one of the major browser vendors. Over a year has passed since my employment and
the suspicion that I’m in a simulation had long faded, so let’s get into what I
could have done better to prepare myself for the job.

Now, because I never thought I’d ever get a job in the browser security space, I
overlooked certain things that companies do to secure their browser. My main
approach to finding bugs is to read as much as I can about browsers and browser
security and then I would manually test out ideas I had. This is a
time-consuming approach, but I found solace in it, sort of like a meditation. At
worst you are left learning about a new Web API and at best you find an
interesting security bug, win-win situation with the only cost being time. This
meant that I never really got into fuzzing, in fact, I had another false
assumption towards that approach – I looked at fuzzing like it was
cryptocurrency mining, thinking I would just run a fuzzer and abuse my CPU until
a worthy crash occurred. A part of me always knew it was the next step in
browser security but perhaps I was too comfortable with my old approach.

Tl;dr: Find your niche, something that excites you and be consistent. You are
already winning by doing what you enjoy.


# Just fix it

As a bug bounty hunter, all I cared about was finding a valid security bug. I
would submit it, maybe argue my case, and then leave and move onto the next bug.
This left me with a gap in my skillset and caused me to miss many opportunities
to learn about how bugs occur under the hood. If you are anything like I am and
want to submit a bug as soon as possible to avoid a potential duplicate, then at
the very least take the time to look at the patch made.

I used to think: “why should I do other people’s jobs for free?!” right? Nope.
The reasons why this is a counter-productive mentality:

1.	You learn the code the buggy part is written in and over time you end up
   learning about (for example) Chromium’s code base and how things work within
   it which translate to you being better at hunting bugs.
2.	Once you figure out the root cause, you can look for other instances of that
   buggy code and potentially turn one bug into multiple bugs.
3.	You might end up finding out your bug is higher severity than you thought
   with slight modification due to that extra insight.
4.	You can get
   [paid to write a patch](https://www.google.com/about/appsecurity/patch-rewards/)
   for the bug you find in Chromium (and others), increasing your total bounty.

One thing I did right was to contribute to conversations happening in the bug
report and Twitter threads. I made sure to read every comment made and those
discussions often have hidden gems. I also did not shy away from disagreeing
with an assessment, given I have evidence of the contrary, I also accepted
denials and moved on amicably. As an employee seeing how things are handled from
the back end, a lot of things made sense. It was surreal to be on the other end
where before I was blind from what was going on.

You see, it is my job now to not only file security bugs but ensure I work with
developers to make sure they get fixed, sometimes it’s straightforward but
others it is not. I must take extra care when filing bugs to make sure I not
only have a minimized reproduction case but to have a root cause analysis and
give my suggestions on how to fix it. Other times, it is a bug reported through
the bug bounty and at times I fill in the gaps of that report and get it to a
state where developers can quickly understand the problem.

The vast majority of bugs reported to us are not real vulnerabilities. Most are
duplicates, by-design, un-reproducible or flat out invalid. A lot of these cases
are not as straightforward so it will trigger a healthy conversation between all
sorts of people depending on the area affected. These conversations contain a
lot of interesting insight and I always look forward to reading the email
threads, as I mentioned before, these types of discussions contain hidden gems.
For me, I found myself often explaining how something is not a vulnerability
than how something is, essentially the opposite of what I used to do but still
similar. So, it’s clear to me my effort in contributing to past bugs I reported
as well as reading all the comments prepared me to handle this part of the job.

Bugs are inevitably going to happen, Edge is growing, and new
features/changes/improvements are being developed constantly and so naturally
the bigger the influx of code being introduced, the higher the number of bugs
that may occur. More specifically the bugs are caused by (applies for both
developers and security teams):
1.	Inexperience. No one is born understanding the ins and outs of the huge Edge
   code base. That takes time and mistakes is how we learn.
2.	Not assuming the worst. When it comes to keeping something secure, a healthy
   level of paranoia helps and always assuming the worst then voicing those
   potential concerns with the security team.
3.	Lack of security education. When I was pursuing my bachelor’s in computer
   science, there were no classes for security. Coding best-practices might be
   mentioned but not the main focus nor are they enough and so educating
   yourself in common pitfalls is important.

One big aspect of the job is educating others (as well as myself) in security.
When a security bug is found, it is an opportunity to discuss it with everyone
involved about the bug found, how it works and how we can prevent it from the
future with automated tests. Security is kept at the forefront of everyone’s
minds by delivering mandatory security training, documenting security best
practices and weekly security tips sent through email.

One thing became apparent early on for me; it is often easier to find a security
bug than it is to write bullet proof code. Therefore, the investment in
developer security awareness is key to securing the browser, none of this can be
achieved without the collaboration with developers where the responsibility for
securing the browser is shared across all teams and not just the security team.

Another part of the job is performing security reviews for every new feature
that are planned to be introduced into Edge. You would think this part of the
job is the most like my browser bug hunting, you would be wrong. Although the
goal is to find bugs in the feature, it also required that I look at actual C++
code, something I rarely did when I hunted for bugs in browsers. Immediately I
had to learn some C++ and get a grasp of the Chromium-Edge code base. It took
some time but with the help of my colleagues I thankfully managed, though I had
wished I’d given more attention to how bugs were fixed rather than whether the
bounty was approved.

Tl;dr: The benefit of digging deeper far outweighs the time you save moving onto
the next thing. So always go deeper and look at the code.

# Doubt

One common theme in my journey is a constant battle with self-doubt. I remember
the time before finding my first bug thinking: “Who am I to find a bug in this
big company?” But I foolishly tried anyway. When offered this job, I caught
myself thinking: “Who am I to work in this big company?” But I did it anyway.

This is a common feeling known as
[impostor syndrome](https://www.techrepublic.com/article/why-58-of-tech-employees-suffer-from-imposter-syndrome/)
and helps to be aware of it to overcome it. Ignore the naysayer in you and try
to do the things you initially don’t think you can, you will be surprised by
yourself. Being somewhat active on Twitter whilst making it a point to only
follow people who are into security and contributing to that community helped
boost up my confidence. I am not talking about getting likes but a word of
encouragement from someone I admire goes a long way. Seeing other people’s work
inspires me and pushes me to be better. This rings true as well by being part of
a team, I am working with people with far more knowledge and skills than I have
and that forces me to become better as I am surrounded by amazing people.

Tl;dr: Be bold and go for it. Be part of a support system, whatever it may look
like.

# Sharing is caring

Most, if not all, bug hunters are self-taught. This means that we learn from the
freely available resources we consume online. Writeups, blogs, papers, slides,
talks and videos discussing security are made with effort by their authors. It
is only natural that you give back and contribute to this open library and play
a role in helping someone achieve their goals. The benefits from doing this far
outweigh the time spent, here are some:

1.	If you are not a native English speaker like me, you get to practice your
   technical writing skills. The ability to effectively communicate a security
   issue is crucial to ensure no one is left confused and things go smoothly.
2.	Writing about a topic helps you re-learn the given topic and most likely you
   will pick up new things you missed the first time.
3.	It contributes to making the internet safer – Your contribution might be a
   key reason for someone finding a different bug and as a result more bugs are
   fixed.
4.	It builds reputation and recognition. People looking to hire or collaborate
   will have you in mind because they might have read some of your work.

Of course, with social media you may fall into the trap of caring more about
likes/views/engagement than the work, I fell into this trap as well. Do not
measure the value of your work by the number of likes, that is fallacious at
best. You can mold social media like twitter to benefit you, just takes time
following the right people and muting the right words.

Yet another preconceived notion I had was that big companies like Microsoft do
not care about what someone writes online. When I wrote about my disappointments
with certain companies, I was just venting to my peers, I did not think anyone
in those companies even cared. Reality is that those blogs/tweets do matter to
the organization, the feedback feature isn’t there for aesthetics, it
legitimately gets looked at. I have witnessed myself how feedback is taken
seriously, and goals are set to satisfy them. This adds another reason for you
to start your own blog/twitter/social as it truly will give you a voice that is
heard.

Tl;dr: Put yourself out there, make yourself known and heard and you will find
yourself part of an amazing community.  

# Pace yourself

When I was bug hunting it was on my time. I did what I wanted to do when I
wanted, if I ever felt burnout coming, I could easily just stop what I’m doing
and do anything else. When I had a day job (before Microsoft) it was also a time
for me to go off of security research and do my job there, so I sort of already
had a balance.

I joined Microsoft during the pandemic and so like a lot of you, I work from
home. As a new employee I wanted to make sure I learn everything I needed to
learn and tried my best to start this new job on a high note, working from home
is also a very new experience I never had to deal with, and it removed that
feeling of leaving the office and sort of hinting to your brain that work mode
is off. The mix of the two lead me to work long hours and work even during the
weekend, I was just so excited and I made a huge mistake in not pacing myself. I
ignored the advice given by teammates regarding burnout and the many work-home
balance messages/reminders given within Microsoft.

I felt the burn. I was mentally exhausted, felt inadequate and just stressed
out. Immediately I took action, stopped working during my weekend, knew when to
end the day and allowed time to decompress. I made sure to sleep at least 7-8
hours and cut back on coffee (just a bit). When I took my first vacation, I made
it a point not to think about work and after coming back my mentality is
completely different. The burnout has subsided and I feel ready to take on the
world.

Tl;dr: Pace yourself, avoid the burnout. It’s a marathon not a sprint.

# How to get into browser security?

Probably one of the most asked questions. For me, I just thought browser bugs
were cool and that motivated me to look for them myself. Other than the low
self-esteem hurdles I talked about earlier, it is as simple as that. You see,
you must do a lot of reading there are no tricks there, reading everything you
can find that has to do with what you are interested in is paramount to
everything else, knowledge is everything.

Here are some links to get you started:

1.	[Chromium code base](https://source.chromium.org/search?q=test) – Contains
   millions of lines of code so look for specific features you’re curious on how
   they work
2.
   [Chromium bug tracker](https://bugs.chromium.org/p/chromium/issues/list?q=Type%3DBug-Security%20RCE&can=1)
   – Specifically bugs filed as security issues, these are filled with
   insightful discussions
3.
   [Chromium security FAQ](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/security/faq.md)
   – Covers a lot of the false positives sent as security issues
4.
   [Chromium severity guidelines](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/security/severity-guidelines.md)
   – This will give you an idea of how security bugs’ severity is determined
   with examples
5.	[Mozilla bug tracker](https://bugzilla.mozilla.org/) – You can find bug
   reports affecting Firefox
6.
   [Mozilla security advisory](https://www.mozilla.org/en-US/security/advisories/)
   – Get the latest Firefox security bug fixes, though for the recent ones
   usually the bugs aren’t yet public
7.	Keep up to date with this blog :-)

Browser bugs can be generally boiled down to two main types:

1.	Memory related bugs – Use after free, buffer overflow and friends, these
   types of bugs
   [make up ~70% of Chromium security bugs](https://www.chromium.org/Home/chromium-security/memory-safety)
   (>=High severity)
    1. Go to ‘about:crashes’ and you can see a list of crashes registered. You
       can find the crash dump at
       `C:\Users\test\AppData\Local\Google\Chrome\User Data\Crashpad\reports`
       (for Chrome)
    2. Get
       [windbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools)
       and open those crash dumps and just execute `!analyze -v`
    3. You can also get Chromium
       [ASAN](https://commondatastorage.googleapis.com/chromium-browser-asan/index.html)
       builds from
       [here](https://commondatastorage.googleapis.com/chromium-browser-asan/index.html),
       which in short allow you to find bugs that otherwise might not trigger on
       normal build and give a nicer stacktrace.
    4. Usually you find these by fuzzing 
2.	Logic bugs – Universal XSS, SOP bypass,
   [UI spoofing](https://microsoftedge.github.io/edgevr/posts/ui-security-thinking-outside-the-viewport/)
   and more are examples of logic bugs that don’t have anything to do with
   corrupting memory
    1. Usually found manually and/or reading source code

I believe I gravitated towards logic bugs naturally since before getting into
security I was mostly into Web technology and so I had already some background
in HTML/JS/CSS/HTTP. So, if you already have some knowledge in Web then focusing
on logic bugs might be an easier transition. On the other hand, if you have more
experience in reverse engineering and/or automation then get into fuzzing and
focus on memory corruption issues. Of course, doing both is doable as well.

Once you have loaded your brain with enough information then you come up with
ideas, try those ideas out and most of them will fail but keep being persistent
and eventually you will hit a security bug.

Tl;dr: Read, read, read, get creative, fail and repeat.

# Automate it all

Fuzzing plays a vital role in any software vendor looking to secure their
application, automation is extremely valuable as it saves a lot of time and can
get a lot of results. For those not aware what fuzzing is, it is the process of
feeding pseudo-randomly generated input into a process (like a browser) and then
detecting if the process crashes. Microsoft invests a lot into fuzzers and it
became quickly apparent that this is a gap in my skills I must fill.

It turns out, my preconceived notion towards fuzzers (to no surprise) was
completely wrong. Fuzzing is a whole ocean in of itself with many different
methodologies and disciplines. My experience with them so far is in the context
of browsers and I’ve only dipped my toes in, but I can already feel the
potential. Here are some tools/fuzzers/generators I’ve messed around with; this
is in no way an extensive list of all things fuzz:

1.	[Libfuzzer](http://llvm.org/docs/LibFuzzer.html) – This is an in-process
   coverage-guided fuzzer. In most basic terms, you set up a function and it
   makes it easy to throw all sorts of input to it and gives back coverage
   information (coverage here means how much code was touched during a given
   input). This requires access to browser source code and some knowledge around
   C++.
2.	[Dharma](https://github.com/MozillaSecurity/dharma) – A grammar based fuzzer.
   This was made by a fellow team member
   [Christoph Diehl](https://twitter.com/posidron) and so I had the pleasure of
   annoying him when I was learning about it. All you do is write a grammar file
   that tells it what a certain Javascript API or HTML element looks like and it
   will generate a whole bunch of testcases you can run in-browser. Does not
   require browser source code but does require JS and HTML knowledge as well as
   some practice on the grammar file syntax.
3. [Playwright](https://github.com/microsoft/playwright)/[Puppeteer](https://github.com/puppeteer/puppeteer)
   – NodeJS API to control the browser. Great tool that gives you a lot of
   control over a browser. Useful if you need to do repetitive tasks using the
   browser. Browser source code not required, light NodeJS background is nice.
4.	[Octo](https://github.com/MozillaSecurity/octo) – Fuzzing library. During
   fuzzing, you will at times need to generate a random string or a random
   number or others. This library gives you easy access to these and it can be
   used inside your NodeJS project or in-browser with a bundled version. Does
   not require browser source code, light NodeJS/JS background is nice.

Tl;dr: Get into automation, expand your horizons and don’t spend too much time
in your comfort zone.

# Reporting browser security bugs

Security bugs will always exist, this is a fact I don’t think will change any
time soon. As you are reading this there is a bug waiting to be found by you, so
go out there and find it.

Assuming you found a bug in Edge then here are some suggestions before reporting:

1.	Does it reproduce on
   [Chrome](https://www.google.com/about/appsecurity/chrome-rewards/)?
    1. Yes – Report it to Chromium instead not Edge
    2. No –
       [Report it to Edge](https://www.microsoft.com/en-us/msrc/bounty-new-edge)
2.	Does it reproduce on Firefox?
    1. Yes – Report it to Edge and
       [Mozilla](https://www.mozilla.org/en-US/security/bug-bounty/) separately
    2. No – Then just report it to Edge
3.	Are you using the latest stable version of Edge?
    1. Go to ‘edge://settings/help’ and see if it’s up to date
    2. Mention that version in your report
4.	Mention what OS you are using (Windows, Mac, Linux, Android or iPhone)
5.	Upload a minimized testcase with your report (avoid sending links to live
   demo)
6.	Upload a short video (I personally use [OBS](https://obsproject.com/)) of the
   behavior
7.	Find a previous publicly disclosed bug in another browser as a precedent

Tl;dr: Bugs are waiting to be found.

# Conclusion

Hope you have found some benefit in my experience so far and learned from some
of my mistakes. The tips I put out are surface level and require you to dig
deeper if you feel confused about something, searching for information is just
as valuable of a skill than anything, so try to refine that skill. If you are
stuck and have questions I am always available to chat on my
[Twitter](https://twitter.com/qab) (DMs open).
