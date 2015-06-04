---
title: "Reflections on Trusting Virtue"
layout: post
category : oss
tags : [oss]
---
There's a very good paper called ["Reflections on Trusting Trust"][1] by Ken Thompson - within that paper he goes on to explain the potential pitfalls of
trusting something you consider trustworthy. As you follow him down the rabbit hole he modifies a C++ compiler to inject a backdoor when compiling UNIX "login",
as well as inject the backdoor injection routine when compiling itself. This results in a compiler that can perpetuate this backdoor without being caught and
provokes an important thought: can you actually trust something that behaves as though it is trustworthy?

Recently we've been rather rudely reminded that maybe we should have paid more attention Ken's words of warning. The story of SourceForge isn't too different
from Ken's C++ compiler.

## SourceForge

SourceForge was once *the* place to set up your open source project. Source control hosting, mailing lists, download mirrors, website hosting - if your project
needed something you could most likely get it from SourceForge. "Find, create, and publish open source software for free" currently heralds all new users to
the site. Everything about the website screams "trustworthy" by virtue of open source.

The problem is that the website can no longer be trusted: [they have turned to adding malware and adware to project binaries.][2] We're seeing the start of
projects [moving away from SourceForge to more trustworthy competitors][3] - clearly a responsible thing to do, given the circumstances. The question is, does
moving away actually achieve anything?

## Your Tombstone

A lot of these projects are leaving tombstones over at SourceForge. The existence of the gimp-win tombstone SourceForge's justification for hijacking the page:
"if you don't care about the project we will continue to 'maintain' it for you." The most obvious solution here is to simply update all your hyperlinks so that
they now point at the new host, again, does this actually achieve anything?

## Hotlinks

For really big and successful projects, probably. They probably have their own web server and deal with their own binary distribution - they haven't been tied
to SourceForge for a very long time.

In the case of SourceForge-hosted binary, updating links is likely to achieve very little. The primary reason is that there is a good chance that the internet
is littered with hotlinks directly to your SourceForge download page, instead of your project domain. How does this happen? Let's take [ruby-lang.org][4] as an
example:

1. Click the big "Download Ruby" button.
2. Presented with a wall of text. Hmmm... "Current stable: Ruby x.x.x" seems about right.
3. A .tar.gz starts downloading, not what I'm looking for.
4. Let me try that "Installation" page.
5. See the "RubyInstaller (Windows)" link and click it.
6. Oh, looks like I need to click *another* "RubyInstaller" link.
7. Click the big "Download" button.
8. Finally, "ruby.x.x.x.exe" is a clickable thing.

The happy path is *7 clicks.* A non-technical user would fail at step 3 and likely YouTube for a tutorial on how to install Ruby. If it's hard to get to your
binaries chances are that 16-year old who made the YouTube video linked directly to your SourceForge download page for the convenience of his viewers. Your bad
landing page has earned a kid some advertising revenue and has earned you a hotlink to SourceForge.

That's just one example of how hotlinks happen, even if your landing page is good (e.g. [mozilla.org][5]) chances someone saw the need to hotlink for one reason
or another - possibly copying a hotlink that someone else posted. The easiest way to deal with the hotlinks is to simply invalidate them by deleting your
project, so does this work?

## SourceForge Controls SourceForge

Even if you delete your project, who's to stop SourceForge from simply re-creating it? They most likely have a backup of the project and can simply re-host it
with their adware binaries. Unless you have a legal way to prevent this (which you probably don't) there's nothing you can do except waste obscene amounts of
time hunting down links and asking webmasters to change them.

## Oh, It's Just SourceForge

You only need to do this once, right?

* The year is 2007, what is your opinion of SourceForge?
* The year is 2015, what is your opinion of Google Code?
* The year is 2020, what is your opinion of CodeHub?

That's not to say the the SourceForge story is the end of all stories, simply be aware that it is one possible outcome. You're placing an immense amount of
trust in these people. In many cases there is nothing you can do, you *have to* place to trust in them.

## What Can I Do?

Unless you have the means to host your own binaries from the get-go, you can only mitigate this risk:

* Become authoritative: have a project domain - even if it points at your project host.
* Reduce hotlinking: user agent sniffing:
  * No more than one click to get the installer for their platform - or -
  * Up-front CLI one-liner to install your thing - or -
  * Up-front instructions on how to install your thing - always -
  * Clearly communicate how to get your official binaries
* Reduce hotlinking: bring in platform maintainers. If someone is building binaries for a platform that you don't support yourselves ask them if you can mirror
  those binaries on the official project website.
* Reduce tombstones: delete project pages if you leave a host.
* Maintain: even if all the bugs have been fixed and all the features are in, maintaining a project is never over. Be prepared to make logistical changes, such
  as moving project hosts.

[1]: https://www.ece.cmu.edu/~ganger/712.fall02/papers/p761-thompson.pdf
[2]: http://www.theregister.co.uk/2015/05/28/sourceforge_accused_of_shackling_gimp_in_kinky_adware/
[3]: https://github.com/tmux/tmux/commit/d2b35e19cdd61d163d26c4babccc1550e72a9623
[4]: https://www.ruby-lang.org/en/
[5]: https://www.mozilla.org/en-US/
