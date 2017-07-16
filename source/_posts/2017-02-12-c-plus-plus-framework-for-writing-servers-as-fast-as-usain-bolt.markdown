---
layout: post
title: "C++ framework for writing servers which are as fast as Usain bolt"
date: 2017-02-12 17:45:25 -0200
comments: true
categories: [cpp, seastar]
keywords: seastar, framework, cpp, shared-nothing, futures, asynchronous, high-performance
description: C++ framework for writing high-performance servers on modern hardware
---

_Prerequisite_: C++

What if I tell you that there's a framework out there that allows you to write
servers that are as fast as the fastest man alive? Let me introduce you to
[Seastar](http://www.seastar-project.org/).
It's a framework written in modern C++ that provides its user with a set of
tools to extract the most from modern hardware.

Look at the chart below which compares Seastar memcached vs stock memcached:
![alt text](http://www.seastar-project.org/img/memcache.png)

And it's not only throughput. It has also proven to improve P99 latency
considerably. Other example of applications that benefit from Seastar are
[ScyllaDB](https://github.com/scylladb/scylla/), a distributed database
compatible with Apache Cassandra, and [Pedis](https://github.com/fastio/pedis/),
a drop-in replacement for Redis.

### What makes Seastar so fast?
Let's go through some of its features and I expect that you will understand its
power by the end of the list. Here we go:

**1.** One of its most interesting features is the [shared-nothing architecture](http://www.seastar-project.org/shared-nothing/).
What exactly is that? Basically, there will be only one thread for each core
(or shard\[1] in Seastar terminology) available to the application. And it also
means that each thread will have its own set of resources (files, memory, etc)
that will not be shared. By doing that, we eliminate the need to use locks
because resources are no longer shared by threads.
Communication between threads must be done explicitly through [message passing](http://www.seastar-project.org/message-passing/),
or else, we would need some locking mechanism.

\[1]: In seastar, a shard is a thread assigned to a CPU core that acts like an
isolated machine.

A common multi-threaded application usually looks like the left picture because
you have threads accessing shared resources, whereas a Seastar application will
look like the right picture because of its shared-nothing design, look:

{% img /images/shared-nothing-pic.png 'shared-nothing' 'images' %}


**2.** Each Seastar thread will have its own scheduler for small asynchronous tasks
(usually stored in [std::function](http://en.cppreference.com/w/cpp/utility/functional/function)),
which is called *The Reactor*. It's important to keep in mind that all tasks
should be asynchronous. That's because if a thread blocks (waiting for I/O, for
example), the reactor will not be able to run other tasks waiting to run.

Let me throw at you another example. What happens if a syscall is called which
involves blocking the calling thread until the requested resource (for example,
sys_read) is satisfied? The CPU would sit idle while waiting for I/O. From the
perspective of a server, it would mean not handling *any* requests. So when
you're writing code for Seastar, you must make sure that you only use
asynchronous API either provided by Linux or Seastar. All I/O in Seastar is
done through asynchronous mechanisms provided by Linux such as aio.

**Seastar API** is very well documented, and it can be found here:
http://docs.seastar-project.org/master/index.html

**3.** Because of the issue mentioned above, Seastar needs some mechanism to make
it easier the task of writing fully-asynchronous code. And that's done using
the concept of [futures and promises](https://en.wikipedia.org/wiki/Futures_and_promises).

Let me explain how future is used in Seastar with code.
Let's say that you want to call a function to sleep for 1 second. Usually, you
would only call a sleep function with 1 as parameter, and then the calling
thread will sleep for 1 second and continue from when it left off.
In Seastar, you shouldn't block the current thread due to performance
reasons, but you can block the task (also known as fiber). So you'd need to
call a Seastar function which promises to wake up the task after 1 second.

```cpp
#include "core/app-template.hh"
#include "core/sleep.hh"
#include <iostream>

int main(int argc, char** argv) {
    app_template app;
    app.run(argc, argv, [] {
        std::cout << "Sleeping... " << std::flush;
        using namespace std::chrono_literals;
        return sleep(1s).then([] {
            std::cout << "Done.\n";
        });
    });
}
```

All functions that need to wait for something (like data from disk or network,
time, etc) return a future type. Those futures can be consumed using
future::then(), which takes a function that will run when the time has come.
In the example above, sleep() returns a future type, which will be waited
using future::then(). So the program above should print 'Done.' after 1 second.
The chain of functions (also known as continuations) can grow as you want.
So after the print, the program could sleep again for N seconds, write to a
file, send a command over the network, whatever you want,
**all asynchronously**!
Please take a look [here](http://www.seastar-project.org/futures-promises/) to
know more about futures and promises in Seastar.


The list is getting long, so I'll tell you quickly what else Seastar provides:

* DPDK support
* userspace TCP/IP stack
* userspace I/O scheduler for fairness among components in the same thread
* and much more!


### Getting started
First of all, you should fork the project, which can be found [here](https://github.com/scylladb/seastar).
Take a look at the README file for instructions on how to install deps and
compile the project.

If you're interested in knowing more about Seastar, I'd advise you to read
[this tutorial](https://github.com/scylladb/seastar/blob/master/doc/tutorial.md)
written by Nadav Har'El and Avi Kivity. If you're willing to delve into some
Seastar apps, please go to:
https://github.com/scylladb/seastar/tree/master/apps

If you need help, send an e-mail to the project's mailing list:
seastar-dev@googlegroups.com


### That's it...
Seastar is very complex and it's very hard to cover many of its aspects in a
single article. At least, I hope I succeeded to help people understand what
Seastar is about and how to get started. I intend to write more articles
about it in the future covering more real-world examples. For example, how to
write a simple server.


Thank you for your time and stay tuned!





