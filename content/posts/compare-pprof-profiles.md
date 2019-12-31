---
title: "Comparing pprof Profiles"
date: 2019-12-31T11:42:58Z
draft: false
---

# What is pprof?

[pprof](https://github.com/google/pprof) is an amazing tool for analyzing and profiling programs. It's invaluable to debugging leaking
goroutines, analyzing memory usage and collecting CPU profiles. In addition, you can generate fancy diagrams in variety of file formats 
(pdf, gif, svg, etc.) for visualizing your programs.

Here's a png example of a heap profile

<a href="/pprof-heap.png" target="_blank"><img src="/pprof-heap.png" style="width: 100%"/></a>

This post will teach you how to set up pprof with Golang and compare profiles so you can better understand your processes and track down memory
leaks.

# Usage

In Go, you can mount http endpoints which serve pprof profiles in protobuf format. The profiles are then used by the `pprof` utility for analysis.

The easiest way to mount the http interface is to import the [net/http/pprof](https://golang.org/pkg/net/http/pprof/) package. This
adds several routes to the `http.DefaultServeMux`.

```
import _ "net/http/pprof"
```

If you don't already have a server running, you'll need to listen and serve

```
go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

Now you can grab profile data under `/debug/pprof/*` routes!

```
wget http://localhost:6060/debug/pprof/goroutine
```

If you prefer, you can get a text representation instead of protobuf data by adding a debug query

```
wget http://localhost:6060/debug/pprof/goroutine?debug=1
```

Note that if you specify `nil` as the handler in http.ListenAndServe, then your server will use the `http.DefaultServeMux`. This is kind of
confusing because importing the pprof package implicitly adds routes to a global, singleton mux.

If you prefer using an explicit mux or a routing system ([gorilla/mux](https://github.com/gorilla/mux),
[go-chi/chi](https://github.com/go-chi/chi), [labstack/echo](https://github.com/labstack/echo), etc.), then you'll have to
explicitly mount the pprof routes like this

```
mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
mux.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine"))
mux.Handle("/debug/pprof/heap", pprof.Handler("heap"))
mux.Handle("/debug/pprof/threadcreate", pprof.Handler("threadcreate"))
mux.Handle("/debug/pprof/block", pprof.Handler("block"))

go func() {
	log.Println(http.ListenAndServe("localhost:6060", mux))
}()
```

# Comparing Profiles

This is super useful! Sometimes you might have a memory leak that you can't track down. One way to diagnose is to take a heap profile,
reproduce the issue and take another profile later. With `pprof` you can essentially diff the profiles and see where memory
is allocated.

But first, how do you even know if you are leaking memory? In general, it's a good practice to observe and alert on some kind of
memory usage indicator.

I like using [prometheus](https://prometheus.io/) for metrics and the Golang client instruments several metrics out of the box. Setting up prometheus and
getting metric data is a bit out of scope for this post but here's an abbreviated prometheus scrape.

```
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 589824
# HELP go_memstats_sys_bytes Number of bytes obtained from system.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 7.2284408e+07
# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
go_threads 11
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 32.93
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1.048576e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 7
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 1.4819328e+07
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.57779196146e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 4.00011264e+08
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes -1
```

Note `process_resident_memory_bytes` which represents how much memory the process is currently using. Setting an alert on this metric is a good practice in case
a change is deployed which inadvertently introduces a memory or goroutine leak. `go_threads` is also good to alert on because leaking gothreads is another common
resource leak, often caused by routines that are blocked on sending or receiving on a shared channel.

OK! So to compare profiles, take an initial snapshot

```
curl localhost:6060/debug/pprof/heap > heap1.data
```

Then reproduce the issue (if you can) or wait some time. Then take another snapshot

```
curl localhost:6060/debug/pprof/heap > heap2.data
```

Now, let's use pprof to analyze the difference[^1] between the two

```
go tool pprof --base heap1.data heap2.data
```

This will drop you into interactive mode of pprof, you can use the `top` command to inspect
the top entries.

```
(pprof) top
Showing nodes accounting for 3.55MB, 100% of 3.55MB total
      flat  flat%   sum%        cum   cum%
    3.55MB   100%   100%     3.55MB   100%  net/http.glob..func5
         0     0%   100%     3.55MB   100%  net/http.(*http2ClientConn).readLoop
         0     0%   100%     3.55MB   100%  net/http.(*http2clientConnReadLoop).processData
         0     0%   100%     3.55MB   100%  net/http.(*http2clientConnReadLoop).run
         0     0%   100%     3.55MB   100%  net/http.(*http2dataBuffer).Write
         0     0%   100%     3.55MB   100%  net/http.(*http2dataBuffer).lastChunkOrAlloc
         0     0%   100%     3.55MB   100%  net/http.(*http2pipe).Write
         0     0%   100%     3.55MB   100%  net/http.http2getDataBufferChunk
         0     0%   100%     3.55MB   100%  sync.(*Pool).Get
```

This gives a few clues that some kind of buffer might be lingering. Double-check that all io.ReadClosers are read and closed!

# Other stuff

You can list the available commands via `help`

```
(pprof) help
  Commands:
    callgrind        Outputs a graph in callgrind format
    comments         Output all profile comments
    disasm           Output assembly listings annotated with samples
    dot              Outputs a graph in DOT format
    eog              Visualize graph through eog
    evince           Visualize graph through evince
    gif              Outputs a graph image in GIF format
    gv               Visualize graph through gv
    kcachegrind      Visualize report in KCachegrind
    list             Output annotated source for functions matching regexp
    pdf              Outputs a graph in PDF format
    peek             Output callers/callees of functions matching regexp
    png              Outputs a graph image in PNG format
    proto            Outputs the profile in compressed protobuf format
    ps               Outputs a graph in PS format
    raw              Outputs a text representation of the raw profile
    svg              Outputs a graph in SVG format
    tags             Outputs all tags in the profile
    text             Outputs top entries in text form
    top              Outputs top entries in text form
    topproto         Outputs top entries in compressed protobuf format
    traces           Outputs all profile samples in text form
    tree             Outputs a text rendering of call graph
    web              Visualize graph through web browser
    weblist          Display annotated source in a web browser
    o/options        List options and their current values
    quit/exit/^D     Exit pprof
```

You can generate visualizations through commands like `png` and `pdf`. You might need to install additional libraries for some 
output formats like DOT which requires graphviz.

```
(pprof) png
Generating report in profile001.png
```

You can also dig deeper into functions using the `list` command and regex matching function names

```
(pprof) list net/http
Total: 10.13MB
ROUTINE ======================== net/http.(*http2ClientConn).readLoop in /usr/local/go/src/net/http/h2_bundle.go
         0     7.62MB (flat, cum) 75.24% of Total
         .          .   8184:					se.Cause = cc.fr.errDetail
         .          .   8185:				}
         .          .   8186:				rl.endStreamError(cs, se)
         .          .   8187:			}
         .          .   8188:			continue
         .     7.62MB   8189:		} else if err != nil {
         .          .   8190:			return err
         .          .   8191:		}
         .          .   8192:		if http2VerboseLogs {
         .          .   8193:			cc.vlogf("http2: Transport received %s", http2summarizeFrame(f))
         .          .   8194:		}
ROUTINE ======================== net/http.(*http2clientConnReadLoop).processData in /usr/local/go/src/net/http/h2_bundle.go
         0     7.62MB (flat, cum) 75.24% of Total
         .          .   8698:			// Values above the maximum flow-control
         .          .   8699:			// window size of 2^31-1 MUST be treated as a
         .          .   8700:			// connection error (Section 5.4.1) of type
         .          .   8701:			// FLOW_CONTROL_ERROR.
         .          .   8702:			if s.Val > math.MaxInt32 {
         .     7.62MB   8703:				return http2ConnectionError(http2ErrCodeFlowControl)
         .          .   8704:			}
         .          .   8705:
         .          .   8706:			// Adjust flow control of currently-open
         .          .   8707:			// frames by the difference of the old initial
         .          .   8708:			// window size and this one.
...
```

This is just a brief introduction but pprof has many more options to tinker with, here are some other great readings
on pprof!

* https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/
* https://blog.golang.org/profiling-go-programs
* https://rakyll.org/custom-profiles/

[^1]: https://github.com/google/pprof/blob/master/doc/README.md#comparing-profiles