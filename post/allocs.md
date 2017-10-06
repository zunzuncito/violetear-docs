+++
title= "Benchmarks and profiling"
date = "2017-03-11T11:03:40+02:00"
description = "0 B/op          0 allocs/op"
tags= ["benchmark", "profiling", "allocs"]
+++


*Go is fast, but when doing things carefully, besides being faster is very light.*

The history about violetear is not much different from many others - a HTTP
router capable of handling any requests.

One of the main constraints found at the beginning was how to deal with static
and dynamic URL's. Let's face it, if you want to replace an important component of
your current architecture you should try to break as less as possible - highly
trained monkeys hate changes. 🐒

For example, the need to support this type of requests:

```html
host.tld/static/8E605AD7-554A-450D-A72D-D0098D336E8E/127.0.0.1
        \_____/\_____________________________________________/
           |                         |
         static                   dynamic

host.tld/static/127.0.0.1/8E605AD7-554A-450D-A72D-D0098D336E8E
        \_____/\_____________________________________________/
           |                         |
         static                   dynamic
```

Which could be represented in the router like:

```go
uuid := `[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}`
router.AddRegex(":uuid", uuid)
router.AddRegex(":ipv4", `^(?:[0-9]{1,3}\.){3}[0-9]{1,3}$`)
router.HandleFunc("/static/:uuid/:ipv4", handleUUID, "PUT")
router.HandleFunc("/static/:ipv4/:uuid", handleIPV$, "GET,HEAD")
```

While implementing this, the use of regular expressions was inevitable, but by
using them where they were not required, they came with a big penalty.

This is how the router was splitting the path for all requests:

{{< highlight go "hl_lines=5" >}}
var splitPathRx = regexp.MustCompile(`[^/ ]+`)

// splitPath returns an slice of the path
func (r *Router) splitPath(p string) []string {
	pathParts := splitPathRx.FindAllString(p, -1)

	// root (empty slice)
	if len(pathParts) == 0 {
		pathParts = append(pathParts, "/")
	}

	return pathParts
}
{{< / highlight >}}

# pprof

Go comes with "batteries included" and is very easy to start profiling the code by the
use of the package [pprof](https://golang.org/pkg/net/http/pprof/).

The first goal was to find the bottlenecks. The following code was used for this:


```go
package main

import (
	"net/http"
	"net/http/pprof"

	"github.com/nbari/violetear"
)

func hello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
}

func main() {
	r := violetear.New()
	r.AddRegex(":word", `^\w+$`)
	r.HandleFunc("/hello", hello, "GET,HEAD")
	r.HandleFunc("/hello/:word/", hello, "GET,HEAD")
	r.HandleFunc("/*", hello, "GET,HEAD")

	// Register pprof handlers
	r.HandleFunc("/debug/pprof/*", pprof.Index)
	r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	r.HandleFunc("/debug/pprof/profile", pprof.Profile)
	r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	r.HandleFunc("/debug/pprof/trace", pprof.Trace)

	http.ListenAndServe(":8080", r)
}
```

Notice the `pprof` handlers:

```go
// Register pprof handlers
r.HandleFunc("/debug/pprof/*", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

For making some random requests, `wrk` + `lua` do a nice combo, this is the
command I used for testing:

```sh
$ wrk -c100 -d10 -t2 -s counter.lua http://0:8080/
```

And the contents of the `counter.lua` script:

```lua
counter = 0

request = function()
    if counter % 100 == 0 then
	    path = "/"
    else
	    path = path .. counter .. "/"
    end
    wrk.headers["X-Counter"] = counter
    counter = counter + 1
    return wrk.format(nil, path)
end
```

This basically will make requests in the following way:

```html
/
/.
/./.
/././.
/./././.
/././././.
/./././././.
/././././././.
/./././././././.
/././././././././.
...
```

It is very basic but allows to test the router with both static and dynamic
handlers.

Once everything is setup, the router should be up, running and listening for
requests, `wrk` is called like mentioned before and in another terminal this
command is used:

```sh
go tool pprof http://0:8080/debug/pprof/heap
```

After running the command you enter into an interactive mode. There I typed
`top` and got this output:

{{< highlight html "linenos=inline,hl_lines=7 10" >}}
(pprof) top
Showing nodes accounting for 4694.50kB, 100% of 4694.50kB total
Showing top 10 nodes out of 25
      flat  flat%   sum%        cum   cum%
 3155.84kB 67.22% 67.22%  3155.84kB 67.22%  regexp.(*bitState).reset /usr/local/Cellar/go/1.9/libexec/src/regexp/backtrack.go
     514kB 10.95% 78.17%      514kB 10.95%  bufio.NewWriterSize /usr/local/Cellar/go/1.9/libexec/src/bufio/bufio.go
  512.62kB 10.92% 89.09%   512.62kB 10.92%  regexp.(*Regexp).FindAllString.func1 /usr/local/Cellar/go/1.9/libexec/src/regexp/regexp.go
  512.03kB 10.91%   100%   512.03kB 10.91%  syscall.anyToSockaddr /usr/local/Cellar/go/1.9/libexec/src/syscall/syscall_bsd.go
         0     0%   100%  3668.46kB 78.14%  github.com/nbari/violetear.(*Router).ServeHTTP /Users/nbari/projects/go/src/github.com/nbari/violetear/violetear.go
         0     0%   100%  3668.46kB 78.14%  github.com/nbari/violetear.(*Router).splitPath /Users/nbari/projects/go/src/github.com/nbari/violetear/violetear.go
         0     0%   100%   512.03kB 10.91%  internal/poll.(*FD).Accept /usr/local/Cellar/go/1.9/libexec/src/internal/poll/fd_unix.go
         0     0%   100%   512.03kB 10.91%  internal/poll.accept /usr/local/Cellar/go/1.9/libexec/src/internal/poll/sys_cloexec.go
         0     0%   100%   512.03kB 10.91%  main.main /Users/nbari/projects/go/src/github.com/nbari/go-sandbox/profile-violetear/profile/main.go
         0     0%   100%   512.03kB 10.91%  net.(*TCPListener).AcceptTCP /usr/local/Cellar/go/1.9/libexec/src/net/tcpsock.go
{{< / highlight >}}

> Notice lines 7 and 10

Still in the interactive mode, I typed: `list splitPath` which helps to show the
lines of code. This was the output:

{{< highlight html "linenos=inline,hl_lines=10" >}}
(pprof) list splitPath
Total: 2.02MB
ROUTINE ======================== github.com/nbari/violetear.(*Router).splitPath in /Users/nbari/projects/go/src/github.com/nbari/violetear/violetear.go
         0     1.02MB (flat, cum) 50.37% of Total
         .          .    261:   return
         .          .    262:}
         .          .    263:
         .          .    264:// splitPath returns an slice of the path
         .          .    265:func (r *Router) splitPath(p string) []string {
         .     1.02MB    266:   pathParts := splitPathRx.FindAllString(p, -1)
         .          .    267:
         .          .    268:   // root (empty slice)
         .          .    269:   if len(pathParts) == 0 {
         .          .    270:           pathParts = append(pathParts, "/")
         .          .    271:   }
(pprof)
{{< / highlight >}}

Was evident that splitting the path by using a regular expression was the most
efficient way to go. I changed the function to something like this:

```go
func (r *Router) splitPath(p string) []string {
        pathParts := strings.FieldsFunc(p, func(c rune) bool {
                return c == '/'
        })
        // root (empty slice)
        if len(pathParts) == 0 {
                return []string{"/"}
        }
        return pathParts
}
```

I created a small bench test:

```go
package violetear

import (
	"net/http"
	"testing"
)

func benchRequest(b *testing.B, router http.Handler, r *http.Request) {
	w := &ResponseWriter{}
	u := r.URL
	rq := u.RawQuery
	r.RequestURI = u.RequestURI()
	b.ReportAllocs()
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		u.RawQuery = rq
		router.ServeHTTP(w, r)
	}
}

func BenchmarkRouterStatic(b *testing.B) {
	router := New()
	router.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {}, "GET,HEAD")
	r, _ := http.NewRequest("GET", "/hello", nil)
	benchRequest(b, router, r)
}

func BenchmarkRouterDynamic(b *testing.B) {
	router := New()
	router.AddRegex(":word", `^\w+$`)
	router.HandleFunc("/test/:word", func(w http.ResponseWriter, r *http.Request) {}, "GET,HEAD")
	r, _ := http.NewRequest("GET", "/test/foo", nil)
	benchRequest(b, router, r)
}
```

And run it using:

    go test -run=^$ -bench=.

The output was a small win of 3 allocation less:

{{< highlight html "linenos=inline,hl_lines=9 14" >}}
$ go test -run=^$ -bench=.
2017/10/02 18:47:23 Adding path: /hello [GET,HEAD]
goos: darwin
goarch: amd64
pkg: github.com/nbari/violetear
BenchmarkRouterStatic-4         2017/10/02 18:47:23 Adding path: /hello [GET,HEAD]
2017/10/02 18:47:23 Adding path: /hello [GET,HEAD]
2017/10/02 18:47:23 Adding path: /hello [GET,HEAD]
 2000000               692 ns/op             616 B/op         10 allocs/op
2017/10/02 18:47:25 Adding path: /test/:word [GET,HEAD]
BenchmarkRouterDynamic-4        2017/10/02 18:47:25 Adding path: /test/:word [GET,HEAD]
2017/10/02 18:47:25 Adding path: /test/:word [GET,HEAD]
2017/10/02 18:47:25 Adding path: /test/:word [GET,HEAD]
 1000000              1256 ns/op             936 B/op         12 allocs/op
PASS
ok      github.com/nbari/violetear      3.380s
{{< / highlight >}}


Before doing the changes this was the output: 11 allocations for static routes
and 14 for dynamic 🐢

{{< highlight html "linenos=inline,hl_lines=9 14" >}}
$ go test -run=^$ -bench=.
2017/10/02 09:56:34 Adding path: /hello [GET,HEAD]
goos: darwin
goarch: amd64
pkg: github.com/nbari/violetear
BenchmarkRouterStatic-4         2017/10/02 09:56:34 Adding path: /hello [GET,HEAD]
2017/10/02 09:56:34 Adding path: /hello [GET,HEAD]
2017/10/02 09:56:34 Adding path: /hello [GET,HEAD]
1000000              1145 ns/op             776 B/op         11 allocs/op
2017/10/02 09:56:35 Adding path: /test/:word [GET,HEAD]
BenchmarkRouterDynamic-4        2017/10/02 09:56:35 Adding path: /test/:word [GET,HEAD]
2017/10/02 09:56:35 Adding path: /test/:word [GET,HEAD]
2017/10/02 09:56:35 Adding path: /test/:word [GET,HEAD]
1000000              1835 ns/op            1096 B/op         14 allocs/op
PASS
ok      github.com/nbari/violetear      3.029s
{{< / highlight >}}

To get more info about the allocations, I mainly used
`pprof` with options like `alloc_space`:

```sh
go tool pprof --alloc_space  http://0:8080/debug/pprof/heap
```

When setting `allocfreetrace=1` to environment variable GODEBUG, you can see
stacktrace where allocation occurs:

```html
allocfreetrace: setting allocfreetrace=1 causes every allocation to be
profiled and a stack trace printed on each object's allocation and free.
```

To build the tests:

```sh
    $ go test -c
```

And to run the tests writing to a file all the output:

```sh
GODEBUG=allocfreetrace=1 ./test.test -test.run=none -test.bench=BenchmarkRouter -test.benchtime=10ms 2>trace.log
```

After doing all this I was available to save up to 90% of the allocations for
static routes, from 11 to 1:

{{< highlight html "linenos=inline,hl_lines=10 15" >}}
$ go test -run=^$ -bench=.
2017/10/02 09:51:47 Adding path: /hello [GET,HEAD]
goos: darwin
goarch: amd64
pkg: github.com/nbari/violetear
BenchmarkRouterStatic-4         2017/10/02 09:51:47 Adding path: /hello [GET,HEAD]
2017/10/02 09:51:47 Adding path: /hello [GET,HEAD]
2017/10/02 09:51:47 Adding path: /hello [GET,HEAD]
2017/10/02 09:51:47 Adding path: /hello [GET,HEAD]
10000000               213 ns/op              16 B/op          1 allocs/op
2017/10/02 09:51:50 Adding path: /test/:word [GET,HEAD]
BenchmarkRouterDynamic-4        2017/10/02 09:51:50 Adding path: /test/:word [GET,HEAD]
2017/10/02 09:51:50 Adding path: /test/:word [GET,HEAD]
2017/10/02 09:51:50 Adding path: /test/:word [GET,HEAD]
1000000              1115 ns/op             816 B/op          7 allocs/op
PASS
ok      github.com/nbari/violetear      3.507s
{{< / highlight >}}


# 0 allocations

Make it happen 😃

{{< highlight html "linenos=inline,hl_lines=10 16" >}}
$ go test -run=^$ -bench=.
2017/10/05 21:57:31 Adding path: /hello [GET,HEAD]
goos: darwin
goarch: amd64
pkg: github.com/nbari/violetear
BenchmarkRouterStatic-4         2017/10/05 21:57:31 Adding path: /hello [GET,HEAD]
2017/10/05 21:57:31 Adding path: /hello [GET,HEAD]
2017/10/05 21:57:31 Adding path: /hello [GET,HEAD]
2017/10/05 21:57:32 Adding path: /hello [GET,HEAD]
10000000               122 ns/op               0 B/op          0 allocs/op
2017/10/05 21:57:33 Adding path: /test/:word [GET,HEAD]
BenchmarkRouterDynamic-4        2017/10/05 21:57:33 Adding path: /test/:word [GET,HEAD]
2017/10/05 21:57:33 Adding path: /test/:word [GET,HEAD]
2017/10/05 21:57:33 Adding path: /test/:word [GET,HEAD]
2017/10/05 21:57:34 Adding path: /test/:word [GET,HEAD]
 2000000               913 ns/op             784 B/op          6 allocs/op
PASS
ok      github.com/nbari/violetear      4.144s
{{< / highlight >}}



The router since always has been behaving quite well, this mainly thanks to Go
itself and well, to the modest traffic it has been handling.

While doing some benchmarks and simulating some DDOS I found some patterns. All
were related to the way the `request.URL.Path` was handled. The thing I noticed
was that the bigger the request path was the more memory allocations were used.
At the end was pretty much making sense since by splitting the path by '/' a
slice was returned with all the possible parts of the path.

For example, for a request like:

```html
/the/quick/brown/fox/jumps/over/the/lazy/dog
```


The router was creating a slice of  10 items:

```go
[ the quick brown fox jumps over the lazy dog]
```

In many cases, this would never be defined as routes and could probably just be
served by a **catch-all(*)** handler.

To fix this, instead of splitting and returning a slice, the path is now parsed.
Each component of the path from left to right is used to find within the set of
defined handlers; so for the same request shown before, the router converts it
to:

```go
key := "the"
path := "/quick/brown/fox/jumps/over/the/lazy/dog"
```

It searches for a possible handler named `the` and if not will try again using
`/the/quick`:

```go
key := "quick"
path := "/brown/fox/jumps/over/the/lazy/dog"
```

Ans so on until a handler matches the `key` or a 404 is returned.

This by principle made the router more efficient but now the challenge was how
to parse the path without creating allocations.

The tricky part was to avoid concatenating strings runes or bytes, by just using
the existing path and return chunks of the slice did the trick, the current code
is this:

{{< highlight go "linenos=inline" >}}
func (t *Trie) SplitPath(path string) (string, string) {
	var key string
	if path == "" {
		return key, path
	} else if path == "/" {
		return path, ""
	}
	for i := 0; i < len(path); i++ {
		if path[i] == '/' {
			if i == 0 {
				return t.SplitPath(path[1:])
			}
			if i > 0 {
				key = path[:i]
				path = path[i:]
				if path == "/" {
					return key, ""
				}
				return key, path
			}
		}
	}
	return path, ""
}
{{< / highlight >}}

The router now is faster and static routes are 0% fat 🚀.

There is still work in progress, who will think that behind a simple router; there
is too much logic evolved. The experience so far has been very pleasant,
learning and improving every day more about go and its niceties.

To make an even better improvement, any tips, comments or a review will be more than welcome 🙏
