---
layout: blog
title: Ebay Access Denied Issue
date: 2025-02-05T22:23:29-06:00
lastMod: 2025-02-05T22:42:38-06:00
categories:
  - websites
tags:
  - website
  - ebay
  - bug
  - browser
  - extensions
  - HoverZoom
description: Finally an issue that has eluded me for years finally comes to a conclusion.
disableComments: true
draft: false
---
For the longest time, I would encounter a strange "Access Denied" message sometimes when browsing Ebay. Which ends up locking me from using the site for a small period of time, sometimes even using a new browser wouldn't work either.

## Access Denied
```
You don't have permission to access "http://www.ebay.com/sch/i.html?" on this server.

Reference #18.1c043517.1738815757.f4abf3b1

https://errors.edgesuite.net/18.1c043517.1738815757.f4abf3b1
```

Eventually I'd get fed up and try searching the internet for an answer only to never find anything from the usual places, Duckduckgo, Google, even Reddit.

That is until tonight when I decided to try my luck again. Low and behold I found this incredible Reddit [thread](https://www.reddit.com/r/Ebay/comments/1bnfhue/access_denied_when_accessing_website/). 

Apparently this whole time the issue was due to a browser plugin that I use often, which is [HoverZoom+](https://github.com/extesy/hoverzoom/) I've been using this handy plugin for years to load images so that I didn't have to click into a image link or post, but never knew it was the cause of the issue. I had always thought it might have been an ad blocker extension or something related to my pfBlockerNG settings in my Router.

Looking at the HoverZoom+ [Github Issues](https://github.com/extesy/hoverzoom/issues/1355), it indeed looks like someone reported it as an issue as of last year without resolution.

So for now I'm happy enough to just disable the Browser extension when browsing Ebay and making this blog post to hopefully help someone else out there that might be running into this issue.

Case Closed.