---
title: Always-use-"local"-to-proxy-video-through-the-server-without-creating-an-account
description: 
published: true
date: 2021-05-23T16:58:12.539Z
tags: 
editor: markdown
dateCreated: 2021-01-28T20:38:58.736Z
---

Download ViolentMonkey for your Browser:  
[Firefox](https://addons.mozilla.org/en-US/firefox/addon/violentmonkey/)  
[Chrome and Chromium](https://chrome.google.com/webstore/detail/violentmonkey/jinjaccalgkegednnccohejagnlnfdag)  
[Others](https://violentmonkey.github.io/get-it/)  

Than add the following script in ViolentMonkey. It will always add `&local=true` to the end of the video URL.

```
// ==UserScript==
// @name        Invidious Proxy automatically
// @match       *://*.invidious.io/watch?v=*
// @run-at      document-start
// @grant       none
// ==/UserScript==


if (!(/[?&]local=/).test(location.search) && !(/[?&]quality=dash/).test(location.search)) {
  location.search += (location.search ? "&" : "?") + "local=true";
}
```

You can also enable this by checking `Proxy videos?` in your preferences.
