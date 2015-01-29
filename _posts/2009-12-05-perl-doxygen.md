---
layout: post
title:  "Doxygen and Perl POD"
date:   2009-12-05 16:27:00
categories: perl doxygen
---

If you have ever dealt with Perl modules on CPAN, you will immediately notice the widespread use of POD to document every thing. And then most POD processors do a simple conversion to HTML (or for that matter many other formats) for proper presentation. pod2html is a very good tool for what it does but when you are maintaining frameworks written entirely in Perl, a simple POD wouldn't be the only documentation that you'll need especially if you want your co-worker to just read it & pick up where you left off. You need to present a little bit more "structure" to your documentation than simply providing description & synopsis for each module. So, when some one refers to framework documentation they are typically not looking for just your method descriptions but a lot more than that like for e.g. an object model or inheritance diagram or the whole frameworks package structure. Sure you can tell them to open up the whole package tree & look at it, but that is just rude - especially with Perl.

This is where documentation generation tools like [Doxygen][doxygen] come in very handy. If you don't know about this tool, I recommend you to go to that site & read up a little about it. Although, if you are a c/c++ shop, it'd be really hard not to come across it or at least consider it. Of course there are other documentation generation tools, but I am a little familiar with doxygen & it was a natural choice. Unfortunately it doesn't support Perl or POD, but fortunately you can feed "filtered" text of your source to it to get the magical output you wanted. There are two projects that I know of that filter Perl source code to be processed by Doxygen - [DoxygenFilter][doxygenfilter] and [DoxyFilt][doxyfilt]. The later is an older project & some how I couldn't figure out how to download the package from its site. Regardless, they both process Perl scripts & at least from their documentation it seemed like they can process POD too. I couldn't try DoxyFilt, but I did try DoxygenFilter & it definitely doesn't process POD (I looked at its modules to confirm). But, the filtered Perl sources were being processed by Doxygen & I was spitting out a nice site of all kinds of wonderful documentation, without any POD info in it. 

So I did what any lazy hacker would do - hack DoxygenFilter to address just my issues. In our group we actually use very basic POD syntax & we enforce simple rules on documentation, like =cut your =head1. These rules meant that I can hack DoxygenFilter really fast to generate necessary input for Doxygen. My changes are attached to this post & remember that this hack is strictly for Perl POD with some arbitrary rules enforced (which works in my group). And it is not optimized, for e.g. I read each Perl script 3 times, but then again my focus was just to get this thing to spit out what I want. I wasn't shooting for programmers hall of fame. 

Anyhoo, the two modules I modified are PerlFilter.pm & Filter.pm which are part of DoxygenFilter package and they are attached to this post. Although I am pretty sure that I made it crystal clear that this hack is for my use & works for my specific situation, I am not guaranteeing that it'll work for you too. Try it at your own risk.

Files: [doxygen-perl.zip][d-p]

[doxygen]: http://www.stack.nl/~dimitri/doxygen/
[doxygenfilter]: http://www.bigsister.ch/doxygenfilter/
[doxyfilt]: http://www.doorways.org/tools/perl/DoxyFilt/index.html
[d-p]: /assets/files/doxygen-perl.zip
