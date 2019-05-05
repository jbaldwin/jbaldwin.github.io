---
layout: post
title: "Lift HTTP Performance"
---

## Tools

For benchmarking liblifthttp we'll use a few tools, `nginx`, `wrk` and a simple lift application that mimics `wrk`'s behavior.  
If you are not familiar with [wrk](https://github.com/wg/wrk) check it out here on github!.  It is a fantastic simple
http benchmark software for hitting your application with as many requests as possible.  There is also [wrk2](https://github.com/giltene/wrk2) 
which allows for testing a steady throughput of requests to check latency of your application at different loads.
I've played around with a few other http benchmarking software, like apache bench (`ab`), but these two are my favorite for
ease of use as well as the simplicity to get the results you want.

## CPU used for benchmarking

While we are trying to compare liblifthttp benchmark with wrk, here is the CPU the benchmarks were run on.

```bash
$ cat /proc/cpuinfo 
...
model name	: Intel(R) Core(TM) i7-7820HQ CPU @ 2.90GHz
...
```

## Baseline Benchmark

For a baseline lets start by using `wrk` and `nginx` to see what a reasonable qps (queries per second) should be for
a lift application to achieve.  Note that while running these benchmarks I like to have `htop` open in a terminal so
I can see how much CPU each process is using.  It can also show when you are using all CPUs available very easy too.

We'll setup a simple hello world HTTP page on `nginx` to fetch as quickly as possible, here are the important snippets of the `nginx` config:

```
worker_processes 1; // we'll adjust this as nginx becomes the bottleneck, but start with 1 for simplicity

events {
    worker_connections 4096; // Might need to adjust this depending on how far we go, 4096 should be plenty for now.
}

server {
    listen 1057 reuseport; // Pick a random port, I was already using 1057 for something else
    server_name localhost;
    root /usr/share/nginx/html;
    location /hello_world.html { }
}
```

And our hello_world.html file put into /usr/share/nginx/html:
```html
<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <h1>Hello world!</h1>
  </body>
</html>
```

Lets reload `nginx`'s conf, I am using Fedora 30 here, your command might vary: `systemctl reload nginx`.

Now lets run `wrk` and see what kind of a baseline performance we can get.  We'll start with 1 thread and 1 connection
and run the test for 30 seconds.

```bash
$ ./wrk  -c 1 -t 1 -d 30s "http://localhost:1057/hello_world.html"
Running 30s test @ http://localhost:1057/hello_world.html
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    33.39us   51.23us   5.97ms   98.72%
    Req/Sec    30.21k     2.12k   32.24k    92.03%
  904965 requests in 30.10s, 4.79GB read
Requests/sec:  30065.34
Transfer/sec:    163.00MB
```

Excellent, we can re-run this test several times, on my CPU I generally get about 30,000 qps.
Lets try adding more connections, 10 should be a good next step.

```bash
$ ./wrk  -c 10 -t 1 -d 30s "http://localhost:1057/hello_world.html"
Running 30s test @ http://localhost:1057/hello_world.html
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   195.97us   74.28us   6.25ms   96.29%
    Req/Sec    51.34k     2.86k   54.92k    90.00%
  1532694 requests in 30.00s, 8.11GB read
Requests/sec:  51088.42
Transfer/sec:    276.98MB
```

Ok, we got a very different result, the bottleneck in the first test was the number of connections we allowed `wrk` to use.
By adding 9 more connections `wrk`'s performance increased by about 40%.  We are not getting a linear
performance increase but we are identifying where the bottlenecks are!  We increased the number of connections for `wrk`, but we haven't increased the number of workers for `nginx, lets try that next.

```bash
sudo vi /etc/nginx/nginx.conf
  -> worker_connections: 1
  -> worker_connections: 2
systemctl reload nginx
```

Lets try the previous test again now with our two `nginx` workers.

```bash
$ ./wrk  -c 10 -t 1 -d 30s "http://localhost:1057/hello_world.html"
Running 30s test @ http://localhost:1057/hello_world.html
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   109.50us  108.56us   6.41ms   97.51%
    Req/Sec    91.40k     5.47k  102.51k    70.00%
  2726966 requests in 30.00s, 14.44GB read
Requests/sec:  90897.41
Transfer/sec:    492.81MB
```

Now we are getting another 43% increase in qps by adding an `nginx` worker process.  Lets try one more test with two
`wrk` worker threads and also bump the connection count to 100.

$ ./wrk  -c 100 -t 2 -d 30s "http://localhost:1057/hello_world.html"
Running 30s test @ http://localhost:1057/hello_world.html
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.20ms    1.27ms  55.98ms   96.39%
    Req/Sec    45.47k     4.90k   56.87k    73.17%
  2713956 requests in 30.00s, 14.37GB read
Requests/sec:  90453.23
Transfer/sec:    490.40MB
```

It seems we've hit a limit with our current setup.  We can take the best qps, ~91,000, with `wrk` settings being 10 connections and 1 worker thread and `nginx` having two worker processes.  I'm sure we could run a lot more tests and get better results, but for this liblifthttp benchmark lets see how close we can get to our simple qps achieved from `wrk` and `nginx`.

## Benchmarking lift

Lift comes with a simple benchmark utility in its examples source directory.  For this test we'll be using this tool.
I would like to note that `wrk` only creates the request body once and re-uses the stringified version of it for each
subsequent request.  Lift also does something similar in this test, however there are differences in the responses.
`wrk` does minimal processing on the response to make it as fast as possible, lift however does a full parse on the
response.  I am pointing out this difference as it will most likely result in a slower speed for lift, however it is
much more realistic in that a real world scenario will require the client to parse http responses.  Lets get started.

As a quick setup we'll build lift first, this will include building the examples as well as the benchmark utility.

```bash
$ cd ~/github/liblifthttp
$ mkdir Release && cd release
$ cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS_RELEASE="-O2" ..
$ make -j4
$ cd bin && ls // produces 'benchmark_simple'
$ ./benchmark_simple 
./benchmark_simple <url> <duration_seconds> <connections> <threads>
```

The CLI for the benchmark is similiar to `wrk` but doesn't use getopts yet, so it expects the parameters in a specific 
order.  Here is our benchmark command: `./benhmark_simple "http://localhost:1057/hello_world.html" 30 1 1`  Lets see how
it does!  I have set `nginx` back to a single worker process to start.

```bash
$ ./benchmark_simple "http://localhost:1057/hello_world.html" 30 1 1
  Thread Stats    Avg
    Req/sec     20155.4
  604662 requests in 30s
Requests/sec: 20155.4
Event loop is no longer accepting requests.
```

In this use case `wrk` is about 33% faster.  Lets try with 10 connections to see the difference.

```bash
$ ./benchmark_simple "http://localhost:1057/hello_world.html" 30 10 1
  Thread Stats    Avg
    Req/sec     40128.8
  1203863 requests in 30s
Requests/sec: 40128.8
```

`wrk` is about 22% faster in this use case.  Here is the breakdown for all of the use cases run.  Note that the CPU
this is running against is 4 core with hyper threading for 8 threads, this is why I stopped the benchmark at 4/4 for
client/server threads.

| Connections | client threads | server threads | wrk qps | lift qps | % diff |
|-------------|----------------|----------------|--------:|---------:|-------:|
| 1           | 1              | 1              | 30,065  | 20,155   |        |
| 10          | 1              | 1              | 50,131  | 39,571   |        |
| 10          | 1              | 2              | 90,370  | 37,337   |        |
| 10          | 2              | 2              | 84,965  |          |        |
| 100         | 2              | 2              | 95,402  |          |        |
| 100         | 2              | 4              | 169,272 |          |        |
| 100         | 4              | 4              | 158,059 |          |        |









