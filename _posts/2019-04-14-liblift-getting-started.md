---
layout: post
title: "Getting started with HTTP Lift library."
---

## What is HTTP Lift library and why create it?

[Lift GitHub Project](https://github.com/jbaldwin/liblifthttp)

The curl library is an amazing piece of software that has been around for a very long time.  Curl has great support for many other protocols, HTTP being one of them, but for Lift the focus is on HTTP only.  Curl has essentially two modes of operation for any kind of request, a synchronous mode via its easy API and an asynchronous mode via is multi mode.  While the easy API is actually fairly easy to use, the multi API is incredibly difficult to get right with its callbacks and requirements for an event loop driver.  That is where Lift comes into the picture, how do we make HTTP requests in C++ via curl multi extremely easy for a programmer while still being efficient in latency and throughput of requests?

Lift HTTP was born out of the need to make thousands of concurrent HTTP requests quickly and efficiently.  Under the hood it is driven by libuv for an event loop and uses libcurl to do the heavy HTTP work.  The design goals around this library:

1. Allow for re-use of request objects (pooling) for efficiency.
2. Simple API for both synchronous and asynchronous requests.
3. C++17 API with easy memory handling for the end user, no raw pointers, no memory leaks by default, low/no copies and moves.

Re-using request objects via a `RequestPool` object is important when the number of requests concurrently executing is important.  Curl on linux likes to hit `/dev/urandom` any time there is a new `curl_easy_handle` created causing a bottleneck in the kernel.  I haven't tested this on other platforms but re-using the `curl_easy_handles` solves this issue.  The `RequestPool` object can `Produce()` a `RequestHandle` for the user of the library to build up any kind of request, sync or async.

The API needs to be simple to use.  Here is a very simple to use synchronous request examples:
```C++
lift::RequestPool pool{};
auto request = pool.Produce("http://www.example.com");

// Note: that "request" is a RAII handle into the actual underlying request object
// and on destruction will return to the pool for re-use.
request->Perform();  // This call is the blocking synchronous HTTP call.

std::cout << request->GetResponseData() << "\n";
```

But what about asynchronous?  Isn't that by default going to be very difficult to use?  Lets take a stab at it by doing the same request above in an async way.
```C++
// Event loops in Lift come with their own RequestPool, no need to provide one.
// Creating the event loop starts it immediately, it spawns a background thread for executing requests.
lift::EventLoop loop{};
auto& pool = loop.GetRequestPool();

// Create the request just like we did in the sync version, but we provide a lambda for on completion.
// Note: that the Lambda is executed ON the Lift event loop thread.  If you want to handle on completion
// processing on this main thread you need to std::move it back via a queue or inter-thread communication.
auto request = pool.Produce(
    "http://www.example.com",
    [](lift::RequestHandle r) { std::cout << r->GetResponseData(); }, // on destruct 'r' will return to the pool.
    10s, // optional time out parameter here
);

// Now inject the request into the event to be executed.  Moving into the event loop is required,
// this passes ownership of the request to the event loop.
loop.StartRequest(std::move(request));

// Block on this main thread until the lift event loop thread has completed the request, or timed out.
while(loop.GetActiveRequestCount() > 0) {
    std::this_thread::sleep_for(10ms);
}

// When loop goes out of scope here it will automatically stop the background thread.  Alternatively you can call Stop().
```

This is by definition more complex and requires more understanding of what is going on.  However, the core API is RAII, requires the user to own memory at specific points starting with the first ownership of producing the new request.  Then relinquishing ownership by moving it into the background Lift event loop thread, and finally re-acquiring ownership via the lambda for that request.  At no point in this example is there any ambiguity on which thread has ownership over the request in question.

While the programmer can still make memory mistakes, nobody is perfect(!), following good modern C++ memory usage and move semantics in Lift's API can make it significantly easier for additional developers to make easy HTTP calls from C++ in an efficient manner.  Give it a try and let me know what you think on the [GitHub](https://github.com/jbaldwin/liblifthttp/issues) project issues.
