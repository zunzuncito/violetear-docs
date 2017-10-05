+++
date = "2017-03-11T11:04:14+01:00"
title = "RequestID"
description = "router.RequestID=\"rid\""

+++

To keep track of the "requests" an existing "request ID" header can be used,
if the header name for example is **X-Appengine-Request-Log-Id** therefore to
continue using it, the router needs to know the name, example:

    router := violetear.New()
    router.RequestID = "X-Appengine-Request-Log-Id"

If the proxy is using another name, for example **RID** then use something like:

    router := violetear.New()
    router.RequestID = "RID"

If ``router.RequestID`` is not set, no "request ID" is going to be added to the
headers. This can be extended using a middleware same has the logger check the
AppEngine example.
