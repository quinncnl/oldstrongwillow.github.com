---
title: Working on the new Proxior
published: false
layout: post
---

*Proxior* aims to be an effective routing HTTP proxy. Basically, it reads HTTP requests and finds the target server or proxy and then sends the request to server, whilst reads responses and sends back to client.

As of the latest version, Proxior does not understand much HTTP protocol. Proxior does not analyze the response header. It does not know whether a connection is consistent or pipelined. And this leads to some intricate problems.

Recently I read the source code of *Polipo*, a well designed HTTP proxy written with no libraries at all. This spurs me to rewrite Proxior. Also, I started using *GoAgentX* a few weeks ago and find some good ideas I could borrow.

The branch name is *2.0* and hosted on my Github.
