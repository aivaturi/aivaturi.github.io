---
layout: post
title:  "Perl Windows - Grab all the system tray icons & get their co-ordinates"
date:   2012-06-26 17:34:00
categories: Perl Windows Automation
---

While automating applications on Windows, every now & then I run in to a situation where I have to find a tray icon & do some mouse operations on it. And for the most part, I overlooked it as it was just a small part of my automation needs. Well, it was time to scratch that small itch as it was bothering me. So with the help of [Sinan on StackOverflow][sinan-so], I ended up writing this script.

This should work on both 32 & 64-bit platforms:

{% gist c01a06be075ef25b86b4 %}
