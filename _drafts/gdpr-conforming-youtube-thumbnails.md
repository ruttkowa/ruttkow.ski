---
layout: post
title: GDPR conforming YouTube Thumbnails
category:
tags:
excerpt_separator: "<!--more-->"
asset_path: "/assets/images/blog/gdpr-conforming-youtube-thumbnails"
image: "/assets/images/blog/gdpr-conforming-youtube-thumbnails/hero.jpg"
---
Linking of embedding YouTube videos in a GDPR conforming way can be quite difficult. As soo as you use the `iFrame` embed option you probably want to wrap it in some sort of "opt in" banner as loading external contentneeds consent of the user.
The easiest alternativ is to just set a `<a href>` link to the corresponding video. 
But I wanted more - so I started to save all of the Thumbnails YouTube generates, reupload them and use them for the link.
As you can image this is quite tedious and time consuming.

That's why I choose to write a little `ruby` script which takes a Youtube URL as the input, downloads the corresponding image from YouTubes static image hosting service (can be found at https://img.youtube.com/vi/ID/0.jpg) and saves it to the current folder. After downloading a copy and paste-able HTML snippet should be sent to `stdoud` as well as the clipboard. 

All of it is a script at `/usr/bin` (or the Mac Equivalent) / somewhere to be loaded in the path. So I can execute the script anywhere just by typing `generateYTThumbnail <YouTube URL>`

<!--more-->