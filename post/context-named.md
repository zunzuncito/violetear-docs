+++
title = "Context & Named parameters"
date = "2017-03-11T11:04:29+01:00"
description = "r.Context().Value(violetear.ParamsKey)"

+++


In some cases there is a need to pass data across
handlers/middlewares, for doing this **Violetear** uses
[net/context](https://golang.org/pkg/context/)

Example:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    "github.com/nbari/violetear"
)

func catchAll(w http.ResponseWriter, r *http.Request) {
    // Get & print the content of named-param *
    params := r.Context().Value(violetear.ParamsKey).(violetear.Params)
    fmt.Fprintf(w, "CatchAll value:, %q", params["*"])
}

func handleUUID(w http.ResponseWriter, r *http.Request) {
    // get router params
    params := r.Context().Value(violetear.ParamsKey).(violetear.Params)
    // using GetParam
    uuid := violetear.GetParam("uuid", r)
    // add a key-value pair to the context
    ctx := context.WithValue(r.Context(), "key", "my-value")
    // print current value for :uuid
    fmt.Fprintf(w, "Named parameter: %q, key: %s",
        params[":uuid"],
        uuid,
        ctx.Value("key"),
    )
}

func main() {
    router := violetear.New()

    uuid := `[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}`
    router.AddRegex(":uuid")

    router.HandleFunc("*", catchAll)
    router.HandleFunc("/:uuid", handleUUID, "GET,HEAD")

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

---

# Duplicated named parameters

In cases where the same named parameter is used multiple times, example:

    /test/:uuid/:uuid/
            |     |
     index  0     1

An slice is created, for getting the values you need to do something like:

        params := r.Context().Value(violetear.ParamsKey).(violetear.Params)
        uuid := params[":uuid"].([]string)

> Notice the ``:`` prefix when getting the named_parameters

Or by using `GetParams`:

        uuid := violetear.GetParams("uuid")

After this you can access the slice like normal:

        fmt.Println(uuid[0], uuid[1])

In case you would only need the second param this could be used:

        uuid := violetear.GetParam("uuid", r, 1)

---

# GetParam & GetParams

This methods help to get an specific param or all the params.

To get a param:

    param := violetear.GetParam("param", r)

When having duplicates, if an index is not specified, it will return the first one, example:

After defining this dynamic routes:

```go
uuid := `[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}`
router.AddRegex(":uuid")
router.HandleFunc("/test/:uuid/:uuid", handleUUID, "GET,HEAD")
```

For this request:

```html
/test/78F204D2-26D9-409F-BE81-2E5D061E1FA1/33A7B724-1498-4A5A-B29B-AD4E31824234
```

To get the first `:uuid`:

```go
uuid := violetear.GetParam("uuid", r)
// uuid == 78F204D2-26D9-409F-BE81-2E5D061E1FA1
```

To get the second `:uuid`:

```go
uuid := violetear.GetParam("uuid", r, 1)
// uuid == 33A7B724-1498-4A5A-B29B-AD4E31824234
```

To get all the params:

```go
uuids := violetear.GetParams("uuid")
// uuids == []string{"78F204D2-26D9-409F-BE81-2E5D061E1FA1", "33A7B724-1498-4A5A-B29B-AD4E31824234"}
```

# Only for go < 1.7

When using go < 1.7:

	import "gopkg.in/nbari/violetear.v1"

> violetear.v1 and violetear.v2 don't support versioning

For been available to use the **Context** ``ctx`` you need to do a type assertion:

    cw := w.(*violetear.ResponseWriter)

To set a key-value pair you need to:

    cw.Set(key, value)

or

    cw.ctx = context.WithValue(cw.ctx, "key", "my-value")


To retrieve a value:

    cw.Get(value)

or

    cw.ctx.Value("key")

> You may need [Type assertions](https://golang.org/ref/spec#Type_assertions): ``cw.Get(value).(string)`` depending on your needs.
