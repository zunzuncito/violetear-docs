+++
title = "NotAllowedHandler"
date = "2017-03-11T11:03:40+01:00"
description = "405 Method Not Allowed"

+++

`NotAllowedHandler` configurable http.Handler which is called when method not allowed:

    405 Method Not Allowed

Example:

    ...

    func my405() http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            http.Error(w, "my custom 405 message", 405)
        })
    }

    func main() {
        router := violetear.New()
        router.NotAllowedHandler = my405()
        ...
