+++
title = "Middleware"
date = "2017-03-11T11:04:30+01:00"
description = "Alice - Middleware Chaining"

+++

Violetear uses [Alice](http://justinas.org/alice-painless-middleware-chaining-for-go/) to handle middleware.

Example:

```go
package main

import (
	"context"
	"log"
	"net/http"

	"github.com/nbari/violetear"
	"github.com/nbari/violetear/middleware"
)

func commonHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("X-app-Version", "1.0")
		next.ServeHTTP(w, r)
	})
}

func middlewareOne(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Executing middlewareOne")
		ctx := context.WithValue(r.Context(), "m1", "m1")
		ctx = context.WithValue(ctx, "key", 1)
		next.ServeHTTP(w, r.WithContext(ctx))
		log.Println("Executing middlewareOne again")
	})
}

func middlewareTwo(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Executing middlewareTwo")
		if r.URL.Path != "/" {
			return
		}
		ctx := context.WithValue(r.Context(), "m2", "m2")
		next.ServeHTTP(w, r.WithContext(ctx))
		log.Println("Executing middlewareTwo again")
	})
}

func catchAll(w http.ResponseWriter, r *http.Request) {
	log.Printf("Executing finalHandler\nm1:%s\nkey:%d\nm2:%s\n",
		r.Context().Value("m1"),
		r.Context().Value("key"),
		r.Context().Value("m2"),
	)
	w.Write([]byte("I catch all"))
}

func foo(w http.ResponseWriter, r *http.Request) {
	panic("this will never happen, because of the return")
}

func main() {
	router := violetear.New()

	stdChain := middleware.New(commonHeaders, middlewareOne, middlewareTwo)

	router.Handle("/", stdChain.ThenFunc(catchAll), "GET,HEAD")
	router.Handle("/foo", stdChain.ThenFunc(foo), "GET,HEAD")
	router.HandleFunc("/bar", foo)

	log.Fatal(http.ListenAndServe(":8080", router))
}
```

> Notice the use or router.Handle and router.HandleFunc when using middleware
you normally would use route.Handle

Request output example:

```sh
$ http http://localhost:8080/
HTTP/1.1 200 OK
Content-Length: 11
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 16:08:18 GMT
Request-Id: GET-1445530098002701428-3
X-App-Version: 1.0

I catch all
```

On the server you will see something like this:

```sh
$ go run test.go
2016/08/17 18:08:42 Adding path: / [GET,HEAD]
2016/08/17 18:08:42 Adding path: /foo [GET,HEAD]
2016/08/17 18:08:42 Adding path: /bar [ALL]
2016/08/17 18:08:47 Executing middlewareOne
2016/08/17 18:08:47 Executing middlewareTwo
2016/08/17 18:08:47 Executing finalHandler
m1:m1
key:1
m2:m2
2016/08/17 18:08:47 Executing middlewareTwo again
2016/08/17 18:08:47 Executing middlewareOne again
```
