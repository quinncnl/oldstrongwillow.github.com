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

We can notice that HTTPConnection includes a pointer to HTTPRequest, and other connection related details. Let's see how Polipo defines a request.

{% highlight c linenos %}
typedef struct _HTTPRequest {
    int flags;
    struct _HTTPConnection *connection;
    ObjectPtr object;
    int method;
    int from;
    int to;
    CacheControlRec cache_control;
    HTTPConditionPtr condition;
    AtomPtr via;
    struct _ConditionHandler *chandler;
    ObjectPtr can_mutate;
    int error_code;
    struct _Atom *error_message;
    struct _Atom *error_headers;
    AtomPtr headers;
    struct timeval time0, time1;
    struct _HTTPRequest *request;
    struct _HTTPRequest *next;
} HTTPRequestRec, *HTTPRequestPtr;
{% endhighlight %}

HTTPRequest is a linked list comprised of pipelined requests to a single server connection.

I can ask many questions about the connection. Below are some representative questions.

#### How a request is received from a client?

*  In *create_listener*, Polipo binds the port and set socket options. At last,

{% highlight c linenos %}

return schedule_accept(fd, handler, data);
{% endhighlight %}

*  *schedule_accept* registers an accept callback. So when a client connects to Polipo, *do_scheduled_accept* is invoked and polipo will save the request to request queue.

{% highlight c linenos %}
  event = registerFdEvent(fd, POLLOUT|POLLIN, 
                            do_scheduled_accept, sizeof(request), &request);

{% endhighlight %}

{% highlight c linenos %}
int
do_scheduled_accept(int status, FdEventHandlerPtr event)
{
    AcceptRequestPtr request = (AcceptRequestPtr)&event->data;
    int rc, done;
    unsigned len;
    struct sockaddr_in addr;

    if(status) {
        done = request->handler(status, event, request);
        if(done) return done;
    }

    len = sizeof(struct sockaddr_in);

    rc = accept(request->fd, (struct sockaddr*)&addr, &len);

    if(rc >= 0)
        done = request->handler(rc, event, request);
    else
        done = request->handler(-errno, event, request);
    return done;
}
{% endhighlight %}

* Use *do_stream* to read the whole header, then call httpClientRequest, as we can see below.

{% highlight bash %}

#0  httpClientRequest (request=0x100106a00, url=0x100106f50) at client.c:725
#1  0x0000000100012d8c in httpClientHandlerHeaders (event=0x100106a00, srequest=0x100106f50, connection=<value temporarily unavailable, due to optimizations>) at client.c:673
#2  0x00000001000120dd in httpClientHandler (status=<value temporarily unavailable, due to optimizations>, event=0x100106f50, request=<value temporarily unavailable, due to optimizations>) at client.c:399
#3  0x000000010000351a in do_scheduled_stream (status=<value temporarily unavailable, due to optimizations>, event=0x100107930) at io.c:240
#4  0x0000000100002caf in eventLoop () at event.c:757
#5  0x000000010000cff6 in main (argc=<value temporarily unavailable, due to optimizations>, argv=<value temporarily unavailable, due to optimizations>) at main.c:165

{% endhighlight %}

*Continuing*
