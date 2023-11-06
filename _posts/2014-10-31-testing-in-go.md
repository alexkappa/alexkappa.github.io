---
layout: post
title: Testing in Go
category: go
author: alex
---

I've been hearing a lot of positive feedback about Google's new programming language for some time now. Interfaces, simplicity, concurrency are some of the things you will definitely hear being praised as you get involved with the community.
{:.lead}

However my favourite thing about the language is something that isn't your typical praising point. Testing is often one of those things you know you need, but keep neglecting because it isn't easy and it's value is not immediatelly clear from a business perspective.

During my short career in making software, I found that people really struggle when it comes to testing their code. I struggled myself when faced with the intricacies of testing object oriented code. The concept of test stubs and mocks was something that is challenging to most newcomers.

Testing in Go is remarkably powerful and it promotes high quality testable code by making it extremely easy to write tests. Now I'm not going to go through the elementary `hello` and `test_hello` introduction, rather start off with something that probably most Go developers have worked on at least once. An http handler.

```go
package main

import "net/http"

func main() {
    http.HandleFunc("/foo", handleFoo)
}

func handleFoo(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("bar"))
}
```

This handler listens for HTTP calls to `/foo` and responds with `"bar"`. Specifically `"bar"` is written to `w` which is an `http.ResponseWriter`. In order to verify what has been written to the response we need to examine `w` after the call to `handleFoo`.

The `http.ResponseWriter` is an interface and is defined as:

```go
type ResponseWriter interface {
	Header() Header
	WriteHeader(http.StatusOK)
	Write([]byte) (int, error)
	WriteHeader(int)
}
```

We could easliy create our own implementation of ResponseWriter to use with our tests, but happily the standard library already has the handy `ResponseRecorder` in the `net/http/httptest` package.

```go
package main

import (
	"testing"
	"net/http"
	"net/http/httptest"
)

func TestHandleFoo(t *testing.T) {
	r, _ := http.NewRequest("", "", nil)
	w := httptest.NewRecorder()

    handleFoo(w, r)

	if w.Body.String() != "bar" {
    	t.Errorf("expected %q but instead got %q", "bar", w.Body.String())
    }
}
```

Of course this example is hardly complex and it doesn't have any dependencies. Typically an HTTP handler would read/write something from a database, log something to a file, or render a template. In that case, we would probably prefer to implement an `http.Handler` instead of an `http.HandlerFunc` which is what `handleFoo` is, so we can define these dependencies.

I'll describe a way to define these dependencies in an upcoming post.

Thanks for reading!
