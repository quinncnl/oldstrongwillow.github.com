---
layout: post
title: Old Pi, New Blood
date: 2013-06-22 20:27:31
---

I used to use Raspbian on my Raspberry Pi, which is good for newbies. However, Debian/Raspbian is still too heavy from my perspective. I ended up choosing the Raspberry Pi ArchLinux because its thin architecture and that they claimed it can boot in less than 10 seconds.

As usual, I configured static IP(took me much longer than I expected), installed transmission, lighttpd, samba, proxior, etc. The php-cgi and php-fpm just can't work properly and I finally switched the blog system to static generated ones, the much hyped Jekyll to be exact.

The writing process is not as straight-forward as wordpress. First, write the markdown post in the _posts folder. Second, build the blog to the web root folder.

It feels quite geeky right?
