su---
layout: post
title: "Broadcasting SDK"
date: 2012-09-25 21:51
comments: true
categories: [SDK, Product, Strategy]
---

Twitch is [flourishing](http://www.businessinsider.com/one-year-later-after-twitchtv-2012-8), but we have projects in the pipeline to change our growth curve.  The existing streaming tools are difficult to use and require expensive gaming rigs or additional hardware.  The goal of our SDK is to integrate streaming into games, dramatically lowering the activation energy to broadcast.  

<!-- more -->

Partnerships have been easier to make than we initially planned for.  The gaming industry is forward thinking and excited about successes in free to play, mobile, and social gaming.  They believe streaming is the next big thing in gaming.  Paradox recently announced their [plans](http://www.businesswire.com/news/home/20120604006646/en) to integrate, and we have several other deals we are very excited about in the works.  

A primary design goal of the SDK is to make it very easy for the game developers to integrate into their games. We have built a simple C-style API with just a handful of functions.  Current count 17 functions.  3-5 Man days is the current estimate from our Beta-Partners as to how long it takes to integrate streaming into a game.

* Handles authentication and obtaining a stream key from the Twitch servers
* Fetches an list of available ingest servers 
* Provides optimal settings (e.g. bitrate, resolution) to game based on available bandwidth
* Game submits its frame buffer to the SDK, which encodes and streams it Twitch
* Capture microphone and speaker audio and mixing them
* Send metadata for discovery: Level, victories, champions, headshots, imagine the possibilites.
* Sets a trace level for debugging purposes

It is critical that the SDK uses as few CPU cycles as possible, thus having minimal impact on the game performance.  We have written our code with performance in mind and have (and will be) tweaking the settings of the encoders to achieve optimal performance.  As an example, we rolled up our sleeves and wrote code to do color space conversion from RGB to YUV using Intel SSE (That's bit by bit). This cut the time to do the color space conversion in half compared to the existing codec.

Our current goal is to get our SDK integrated into one of our partner's beta.  Getting our product to the point where developers are willing to push it to their users is productive, and we are going to learn a ton once we are in the hands of real users.  Exciting times.