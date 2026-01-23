---
title: "Claude the Hacker"
date: 2026-01-23
draft: true
description: "Integrating Claude into a pentesting workflow"
tags: ["meta"]
categories: ["general"]
toc: false
---

With the advent of these extremely powerful CLI coding agents, I decided to test out and see how well something like Claude performs in the context of independent security researching (poking around vulnerable systems).

I began with some simple recon. I went over to [shodan](shodan.io), and looked around for some vulnerable systems running Telnet with root already logged in (not going to post about how to do this, but it's not hard).

I found many vulnerable IPs, but chose one in Hong Kong at random. After manually testing whether or not I could remotely access the system and succeeding, it was now time to see if Claude could do the same.

When trying it out, the first issue I ran into was that the connection immediately dropped. Claude was able to connect without issue, but since telnet expects continuous real-time reponses from the client, it drops the connection due to Claude not being able to do that.

There is an extremely easy workaround though, and it was suggested by none other than Claude itself: piping commands through netcat. (By the way, it remains incredibly easy to gaslight AI into doing these things)

This inevitably worked, and from there it's basically a free-for-all. I instructed Claude to crawl through the system and take note of any sensitive files, misconfigurations, vulnerabilities, and anything that seemed odd. I half-expected Claude to tell me that it wasn't allowed to do so, but instead it spent the next five minutes exploring everything and then gave me a detailed report. A report that gave me a lot to work with...

This lowers the barrier of entry for threat actors significantly. Claude explored the system, created multiple Python scripts to help automate the process, and investigated things that seemed out of place like strange ports.

What would have taken me a couple of hours before was accomplished in little under ten minutes. I began with an IP address sourced from shodan, and a little bit later I ended up with a detailed analysis of the system, allowing me to instead focus on crafting an exploit (in theory). 95% of the recon process was able to be done by Claude, and if I so wished I could probably get Claude to work directly with shodan and skip that part as well.