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

```json
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
    Latency    76.99us   76.64us   6.50ms   95.76%
    Req/Sec   119.68k     9.93k  141.78k    66.67%
  3571309 requests in 30.00s, 18.91GB read
Requests/sec: 119036.92
Transfer/sec:    645.37MB
```

Ok, we got a very different result, the bottleneck in the first test was the number of connections we allowed `wrk` to use.
By adding 9 more connections `wrk`'s performance increased by 290%, almost three times faster.  We are not getting a linear
performance increase but we are identifying where the bottlenecks are!  We increased the number of connections for `wrk`, but 
we haven't increased the number of workers for `nginx, lets try that next.

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

It appears we had a degreation in performance by adding another worker to `nginx` in the same scenario, why could that be?
Perhaps there are not enough connections to saturate `nginx` again or there is overhead in `nginx` across its workers.
Lets try bumping up the number of connections again in `wrk` and see if we get a better result.  We'll also try running
`wrk` with two worker threads as well.

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

...

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

It seems we've hit a limit with our current setup.  We can take the best qps, 119036, with `wrk` settings being 10 connections and 1 worker thread.
`nginx running with a single worker processor.  I'm sure we could run a lot more tests and get better results, but for this liblifthttp benchmark
lets see how close we can get to our simple qps achieved from `wrk` and `nginx`.





