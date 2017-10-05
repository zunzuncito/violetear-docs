+++
date = "2017-03-11T10:11:01+00:00"
description = ""
title = "How it works"

+++

The router is capable of handle any kind or URI, static,
dynamic or catchall and based on the
[HTTP request Method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)
accept or discard the request.

For example, suppose we have an API that exposes a service that allows to ping
any IP address.

To handle only `GET` requests for any IPv4 addresss:

    http://api.violetear.org/command/ping/127.0.0.1
                            \______/\___/\________/
                                |     |      |
                                 static      |
                                          dynamic

The router ``HandlerFunc``  would be:

    router.HandleFunc("/command/ping/:ip", ipHandler, "GET")

For this to work, first the regex matching ``:ip`` should be added:

    router.AddRegex(":ip", `^(?:[0-9]{1,3}\.){3}[0-9]{1,3}$`)

Now let's say you also want to be available to ping ipv6 or any host:

    http://api.violetear.org/command/ping/*
                            \______/\___/\_/
                                |     |   |
                                 static   |
                                       catch-all

A catch-all could be used and also a different handler, for example:

    router.HandleFunc("/command/ping/*", anyHandler, "GET, HEAD")

The ``*`` indicates the router to behave like a catch-all therefore it
will match anything after the ``/command/ping/`` if no other condition matches
before.

Notice also the `GET`, `HEAD`, that indicates that only does HTTP methods will
be accepted, and any other will not be allowed, router will return a:

    405 Method Not Allowed

To customise this reponse, the [`NotAllowedHandler`](/post/notallowedhandler/)
can be used, it is a configurable [http.Handler](https://golang.org/pkg/net/http/#Handler)
which is called when method not allowed.
