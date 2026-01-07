---
title: "Nixie Clock: Software"
date: 2025-11-23
description: "a nixie based clock"
thumbnail: "/featured.jpeg"
tags: ["Nixie", "Clock"]
series: ["Nixie clock"]
series_order: 2
---

Overview
========

I built this project a while ago and I just got around to write it down on my blog. I can see the marked difference in how I wrote back then, and in how I do it now.

In an attempt to keep things tidy I broke the code in different files,


Let's start by saying that the Arduino IDE is not a good choice to write code. Just here I tried dividing the various functions in multiple files, and it's already difficult to keep track of everything, even with a total of around a thousand lines of code. 

What I used the last time I wrote Arduino code was to use VSCode and use the PlatformIO plugin.

For this specific project I used an ATtiny1616, which is programmable trough PlatformIO. To do so it uses the megaTinyCore, more info regarding the set-up of the core [here](https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/PlatformIO.md).

The 