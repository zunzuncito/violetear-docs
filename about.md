+++
date = "2017-03-11T09:57:00+01:00"
title = "violetear"

+++

# What is violetear?

An HTTP router, extending the capabilities of the default
[go mux](https://golang.org/pkg/net/http/#ServeMux)

---

## Features

* 0 memory allocations for static routes.
* Common context between middleware.
* [Context & Named](/posts/context-named.md) parameters
* Easy [middleware](/post/middleware) (satisfies the http.Handler interface).
* HTTP/2 native support [Push Example](https://gist.github.com/nbari/e19f195c233c92061e27f5beaaae45a3)
* Support for [static and dynamic](/post/usage) routing.
* Trace [Request-ID](/post/requestid) per request.
* Versioning `application/vnd.*`
