---
title: Estimating the Testing Matrix for Audio Plugins
layout: post
tags: programming
---

Audio plugins need to work in a wide range of environments. 

[Floe](https://floe.audio) targets the following environments:
- Operating systems: Windows, macOS, Linux (dev only for now)
- Plugin formats: CLAP, VST3, AUv2 (macOS only)

Audio plugins are loaded by host software. Each host uses the audio plugin API in slightly different ways.

Let's estimate the total number of environments by making some back-of-the-envelope estimations.

First, let's say there are 20 different hosts. No exact number exists, but 20 covers all of the major hosts: Cubase, Ableton Live, Logic, Garageband, Studio One, FL Studio, Tracktion, Bitwig, Reaper, Reason, Cakewalk; and also the minor hosts: Zrythm, Carla, MultitrackStudio, LMMS, qtrackor, Soundbridge, etc.

Next, not all hosts support all OSs and all plugin formats. So we should skew our estimation by a certain amount. 0.2 seems like a reasonable, conservative skew factor. This would assume that on average, each host only supports 20% of the `OS` x `plugin format` matrix. In reality, some support more and some support less.

```
OS = 3, formats = 3, hosts = 20, skew = 0.2. 
OS x formats x hosts x skew = 36
```

36 is a lot of environments!

This isn't to suggest that each environment is totally different. Plugin standards exist for a reason so that you shouldn't have to write code handling 36 environments individually. Instead, you should write for the plugin format and OS and this should work everywhere. 

The reality is a bit messier though. I've seen lots of issues only arise in certain DAW/OS combinations, and I've seen other developers face this issue too. This is typically due to the particular sequence of calls that a host makes to the plugin.

In conclusion, this estimation might help work out how much testing is needed, and how careful we need to be regarding correctly implementing plugin standards. Floe strives for professional reliability in all environments so this has been on my mind.
