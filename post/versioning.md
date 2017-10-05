+++
title = "Versioning"
date = "2017-03-11T11:04:28+01:00"
description = "application/vnd.acme.v2"

+++

**Violetear** handle versions by using the [`Accept: application/vnd.*`](https://en.wikipedia.org/wiki/Media_type#Naming)
header.

This means that the URL won't change, therefore instead of specifying the version on the URL something like this:

```html
https://acme.tld/api/v2/foo
```

It can always be:

```html
https://acme.tld/api/foo
```

But based on the request `Accept: application/vnd.*` header the corresponding
version will be served:

    https://acme.tld/api/foo
    ===>
    GET /api/foo HTTP/1.1
    Accept: application/vnd.acme.v2

---

## Using the Fragment identifier `#`

To define what version to use, the `#` [fragment identifier](https://en.wikipedia.org/wiki/Fragment_identifier) is used when
declaring the routes.

The `versionHeader` constant is set to:

    application/vnd.

This means that when defining a version only the string after `application/vnd.` is required:

    application/vnd.[version]


| Request Accept header| route version in violetear|
|----------------------|-------------------|
|application/vnd.acme.v2 | #acme.v2 |
|application/vnd.acme.v2+json| #acme.v2+json|
|application/vnd.acme.v2.raw+json| #acme.v2.raw+json|

Example:

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/nbari/violetear"
)

func handleHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi,  %s!", r.URL.Path[1:])
}
func handleHelloV2(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "version 2")
}

func main() {
	router := violetear.New()

	router.HandleFunc("/hello", handleHello, "GET, HEAD")
	router.HandleFunc("/hello#violetear.v2", handleHelloV2, "GET, HEAD, POST")

	log.Fatal(http.ListenAndServe(":8000", router))
}
```

In this example, there are 2 different handlers for the `/hello/` request `handleHello` and `handeHelloV2`:

	router.HandleFunc("/hello", handleHello, "GET, HEAD")
	router.HandleFunc("/hello#violetear.v2", handleHelloV2, "GET, HEAD, POST")

To serve using the `handleHelloV2`, the request needs to contain the header:

    Accept: application/vnd.violetear.v2

if not, will take as default the handler `handleHello`

---

Example of how to send the header using http or curl

Using the http client:

    $ http 0:8000/hello "Accept: application/vnd.violetear.v2"

Using curl:

    $ curl 0:8000/hello -H "Accept: application/vnd.violetear.v2"
