+++
title = "PanicHandler"
date = "2017-03-11T11:03:40+01:00"
description = "50X"

+++

`PanicHandler` function to handle panics.


Example:

```go
func myPanicHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        http.Error(w, "custom panic message", 500)
    })
}

router := violetear.New()
router.PanicHandler = myPanicHandler()
```
