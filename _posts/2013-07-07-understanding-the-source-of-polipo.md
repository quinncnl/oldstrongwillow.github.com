---
layout: post
---

## Overview

The entry point is in main.c. eventLoop() in event.c infinitely loops polling for activities.

In order to understand the procedure, I added many *puts* and *printf* to track the code. Besides, adding some breakpoints in gdb can also be very helpful.


## Event flow

Polipo use the effective IO multiplexing *poll* to listen on IO events. According to its homepage,

> Polipo will use HTTP/1.1 pipelining if it believes that the remote server supports it, whether the incoming requests are pipelined or come in simultaneously on multiple connections (this is more than the simple usage of persistent connections, which is done by e.g. Squid);

For example, if broswer A is requesting http://xxx/abc, while broswer B is requesting http://xxx/xyz, Polipo will pipeline the requests and use only one connection to server xxx. The machanism is simple. *expireServersHandler* is just for that.

{% highlight c linenos %}

/**
When a server has sent all response, keep it alive for a while in case another client requests the same server.
*/

static int
expireServersHandler(TimeEventHandlerPtr event)
{
    HTTPServerPtr server, next;
    TimeEventHandlerPtr e;
    server = servers;
    while(server) {
        next = server->next;
        if(httpServerIdle(server) &&
           server->time + serverExpireTime < current_time.tv_sec)
            discardServer(server);
        server = next;
    }
    e = scheduleTimeEvent(serverExpireTime / 60 + 60, 
                          expireServersHandler, 0, NULL);
    if(!e) {
        do_log(L_ERROR, "Couldn't schedule server expiry.\n");
        polipoExit();
    }
    return 1;
}

{% endhighlight %}


Time event handlers such as *expireServersHandler*, *httpTimeoutHandler*, *httpClientDelayed*, *dnsTimeoutHandler* are scheduled in a queue using

> TimeEventHandlerPtr
> scheduleTimeEvent(int seconds,
>                  int (*handler)(TimeEventHandlerPtr), int dsize, void *data)

If no time events registered, Polipo will be blocked polling for new requests from clients. If there is some time events waiting in the queue, then Polipo will check if any of the events timed out. As the following code snippet manifests.

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

## Connection

First of all, let's see how Polipo defines HTTP connection.

{% highlight c linenos %}

typedef struct _HTTPConnection {
    int flags;
    int fd;
    char *buf;
    int len;
    int offset;
    HTTPRequestPtr request;
    HTTPRequestPtr request_last;
    int serviced;
    int version;
    int time;
    TimeEventHandlerPtr timeout;
    int te;
    char *reqbuf;
    int reqlen;
    int reqbegin;
    int reqoffset;
    int bodylen;
    int reqte;
    /* For server connections */
    int chunk_remaining;
    struct _HTTPServer *server;
    int pipelined;
    int connecting;
} HTTPConnectionRec, *HTTPConnectionPtr;
{% endhighlight %}

*Continuing*
