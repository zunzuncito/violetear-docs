+++
title = "Usage"
description = "Basic example"
date = "2017-03-11T11:05:52+01:00"

+++

Basic example:

```go
package main

import (
    "github.com/nbari/violetear"
    "log"
    "net/http"
)

func catchAll(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("I'm catching all\n"))
}

func handleGET(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("I handle GET requests\n"))
}

func handlePOST(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("I handle POST requests\n"))
}

func handleUUID(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("I handle dynamic requests\n"))
}

func main() {
    router := violetear.New()
    router.LogRequests = true
    router.RequestID = "Request-ID"

    uuid := `[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}`
    router.AddRegex(":uuid", uuid)

    router.HandleFunc("*", catchAll)
    router.HandleFunc("/method", catchALL) // will handle all methods but GET, POST
    router.HandleFunc("/method", handleGET, "GET")
    router.HandleFunc("/method", handlePOST, "POST")
    router.HandleFunc("/:uuid", handleUUID, "GET,HEAD")

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

Running this code will show something like this:

```sh
$ go run test.go
2015/10/22 17:14:18 Adding path: * [ALL]
2015/10/22 17:14:18 Adding path: /method [GET]
2015/10/22 17:14:18 Adding path: /method [POST]
2015/10/22 17:14:18 Adding path: /:uuid [GET,HEAD]
```

Using ``router.Verbose = false`` will omit printing the paths.

> test.go contains the code show above

Testing using curl or [http](https://github.com/jkbrzt/httpie)

Any request 'catch-all':

```sh
$ http POST http://localhost:8080/
HTTP/1.1 200 OK
Content-Length: 17
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 15:18:49 GMT
Request-Id: POST-1445527129854964669-1

I'm catching all
```

A GET request:

```sh
$ http http://localhost:8080/method
HTTP/1.1 200 OK
Content-Length: 22
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 15:43:25 GMT
Request-Id: GET-1445528605902591921-1

I handle GET requests
```

A POST request:

```sh
$ http POST http://localhost:8080/method
HTTP/1.1 200 OK
Content-Length: 23
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 15:44:28 GMT
Request-Id: POST-1445528668557478433-2

I handle POST requests
```

A dynamic request using an [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) as the URL resource:

```sh
$ http http://localhost:8080/50244127-45F6-4210-A89D-FFB0DA039425
HTTP/1.1 200 OK
Content-Length: 26
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 15:45:33 GMT
Request-Id: GET-1445528733916239110-5

I handle dynamic requests
```

Trying to use POST on the ``/:uuid`` resource will cause a
`Method not Allowed 405` this because only `GET` and `HEAD`
methods are allowed:

```sh
$ http POST http://localhost:8080/50244127-45F6-4210-A89D-FFB0DA039425
HTTP/1.1 405 Method Not Allowed
Content-Length: 19
Content-Type: text/plain; charset=utf-8
Date: Thu, 22 Oct 2015 15:47:19 GMT
Request-Id: POST-1445528839403536403-6
X-Content-Type-Options: nosniff

Method Not Allowed
```


## ALL

If no request method is defined ``ALL`` methods will be defined.


### static routes over regular expressions

Static routes always will win, have preference over regular expressions, for example:

```go
router.AddRegex(":uuid", `\w+`)
router.HandleFunc("/method", methodGET, "GET, HEAD")
router.HandleFunc("/:any", methodALL)
```

All request to `/method` using methods `GET` and `HEAD` will be handled by
`methodGET` and all others by method `/methodALL`

Example using same endpoint for multiple methods:

```go

package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/nbari/violetear"
)

func methodGET(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "methodGET request method: %s", r.Method)
    log.Printf("method = %s\n", r.Method)
}

func methodPOST(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "methodPost request method: %s", r.Method)
}

func methodALL(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "methodAll request method: %s", r.Method)
}

func methodRX(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "methodRX request method: %s", r.Method)
}

func main() {
    router := violetear.New()
    router.LogRequests = true
    router.Verbose = true

    router.AddRegex(":any", `\w+`)

    router.HandleFunc("/method", methodGET, "GET, HEAD") // will handle only GET and HEAD
    router.HandleFunc("/method", methodPOST, "POST")     // will handle only POST
    router.HandleFunc("/method", methodALL)              // all but GET, HEAD, POST
    router.HandleFunc("/:any", methodRX)                 // will handle any method

    log.Fatal(http.ListenAndServe(":8080", router))
}
```
