+++
title = "NotFoundHandler"
date = "2017-03-11T11:03:43+01:00"
description = "404 Not Found"

+++

For defining a custom ``http.Handler`` to handle **404 Not Found** example:

    ...

    func my404() http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            http.Error(w, "my custom 404 message", 404)
        })
    }

    func main() {
        router := violetear.New()
        router.NotFoundHandler = my404()
        ...
