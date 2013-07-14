---
layout: post
title: Understanding the Source of Polipo (Not Completed)
---

## Overview

The entry point is in main.c. eventLoop() in event.c infinitely loops polling for activities.

In order to understand the procedure, I added many *puts* and *printf* to track the code. Besides, adding some breakpoints in gdb can also be very helpful.


## Event flow

Polipo use the effective IO multiplexing *poll* to listen on IO events. According to its homepage,

> Polipo will use HTTP/1.1 pipelining if it believes that the remote server supports it, whether the incoming requests are pipelined or come in simultaneously on multiple connections (this is more than the simple usage of persistent connections, which is done by e.g. Squid);

For example, if broswer A is requesting http://xxx/abc, while broswer B is requesting http://xxx/xyz, Polipo have the ability to pipeline the requests and use only one connection to server xxx. In order to achieve this, Polipo introduces a delayed reading mechanism.

Time event handlers such as *expireServersHandler*, *httpTimeoutHandler*, *httpClientDelayed*, *dnsTimeoutHandler* are scheduled in a queue using

> TimeEventHandlerPtr
> scheduleTimeEvent(int seconds,
>                  int (*handler)(TimeEventHandlerPtr), int dsize, void *data)

If no time events registered, then Polipo will be blocked polling for new requests from clients. If there is some time events in waiting in the queue, 

{% highlight c linenos %}

// event.c eventLoop


if(sleep_time.tv_sec == -1) {
  //if timeEventQueue has no items, poll for non-time events
    rc = poll(poll_fds, fdEventNum, 
              diskIsClean ? -1 : idleTime * 1000);
} else if(timeval_cmp(&sleep_time, &current_time) <= 0) {
  //sleep_time <= current_time
  //Timeout and start to run timed out events
    runTimeEventQueue();
    continue;
} else {
  //timeEventQueue has items and has not timed out
  //refresh current time
    gettimeofday(&current_time, NULL);
    if(timeval_cmp(&sleep_time, &current_time) <= 0) {
	  //refreshed and find timeout
        runTimeEventQueue();
        continue;
    } else {
	  //still sleeping
        int t;
        timeval_minus(&timeout, &sleep_time, &current_time);
        t = timeout.tv_sec * 1000 + (timeout.tv_usec + 999) / 1000;

		//blocking here for requests
		printf("blocking here\n");
        rc = poll(poll_fds, fdEventNum,
                  diskIsClean ? t : MIN(idleTime * 1000, t));
    }
}

{% endhighlight %}
