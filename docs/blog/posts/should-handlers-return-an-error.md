---
title: "Should Handlers Return an Error?"
description: "A practical look at how returning errors from HTTP handlers — instead of writing responses inline — leads to cleaner, more maintainable Go services."
date: 2026-08-22
categories:
  - Go
  - Web Development
  - Best Practices
tags:
  - go
  - http
  - error-handling
  - gin
  - clean-code
authors:
  - nazrawi
---

# Should Handlers Return an Error?

I was watching the **Backend Banter** podcast — specifically the episode featuring AnthonyGG — and one of the topics they touched on really stuck with me: should HTTP handlers return an error?

[![Talking Go with the Go God feat. AnthonyGG | Backend Banter 055](https://img.youtube.com/vi/00XOiggcQ8Q/maxresdefault.jpg)](https://www.youtube.com/watch?v=00XOiggcQ8Q&t=1588 "Talking Go with the Go God feat. AnthonyGG | Backend Banter 055")

Let's dig into why this pattern is more powerful than it looks.

---

## The `error` Interface Is Just an Interface

In Go, `error` is nothing more than a single-method interface:

```go
type error interface {
    Error() string
}
```

!!! info "Official Reference"
    See the Go team's own write-up: [Error handling and Go](https://go.dev/blog/error-handling-and-go)

Because it's just an interface, **you can implement it with any struct you want** — including one that carries an HTTP status code:

```go
// APIError carries both an HTTP status and a human-readable message.
type APIError struct {
    Status int    `json:"-"`
    Detail string `json:"error"`
}

func (e APIError) Error() string {
    return e.Detail
}
```

`APIError` satisfies the `error` interface, so it can be returned anywhere a plain `error` is expected — but it carries far more context.

---

## The Problem with Standard Handlers

The conventional `http.HandlerFunc` signature looks like this:

```go
func myHandler(w http.ResponseWriter, r *http.Request)
```

It returns nothing. That means **every error case must be handled inline** — you're writing response logic directly inside each handler, over and over:

```go
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    id := r.URL.Query().Get("id")
    if id == "" {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": "missing user ID"})
        return
    }
    // ... more inline error handling below
}
```

This scatters HTTP error formatting across every handler in your codebase. It's repetitive, hard to keep consistent, and easy to get wrong.

---

## The Fix: Handlers That Return an Error

The idea is simple — define a custom handler type that **returns an error**:

```go
type APIHandler func(w http.ResponseWriter, r *http.Request) error
```

Then wrap it in a higher-order function that converts the returned error into an HTTP response. This pattern is sometimes called a **handler decorator** or **MakeHTTPHandler**:

```go
func MakeHTTPHandler(h APIHandler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if err := h(w, r); err != nil {
            var apiErr APIError

            switch {
            case errors.As(err, &apiErr):
                writeJSON(w, apiErr.Status, apiErr)

            case errors.Is(err, ErrNotFound):
                writeJSON(w, http.StatusNotFound, APIError{Detail: err.Error()})

            case errors.Is(err, ErrUnauthorized):
                writeJSON(w, http.StatusUnauthorized, APIError{Detail: err.Error()})

            default:
                log.Printf("internal server error: %v", err)
                writeJSON(w, http.StatusInternalServerError, APIError{Detail: "internal server error"})
            }
        }
    }
}

func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}
```

!!! tip "Why This Works"
    `errors.As` unwraps the error chain looking for a concrete `APIError`. `errors.Is` checks for sentinel errors like `ErrNotFound`. The `default` case is your safety net — it logs the raw error server-side without leaking internals to the client.

---

## Full Example: Standard `net/http`

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "net/http"
)

var (
    ErrNotFound     = errors.New("resource not found")
    ErrUnauthorized = errors.New("unauthorized access")
)

type APIError struct {
    Status int    `json:"-"`
    Detail string `json:"error"`
}

func (e APIError) Error() string { return e.Detail }

type APIHandler func(w http.ResponseWriter, r *http.Request) error

func MakeHTTPHandler(h APIHandler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if err := h(w, r); err != nil {
            var apiErr APIError
            switch {
            case errors.As(err, &apiErr):
                writeJSON(w, apiErr.Status, apiErr)
            case errors.Is(err, ErrNotFound):
                writeJSON(w, http.StatusNotFound, APIError{Detail: err.Error()})
            case errors.Is(err, ErrUnauthorized):
                writeJSON(w, http.StatusUnauthorized, APIError{Detail: err.Error()})
            default:
                log.Printf("Internal Server Error: %v", err)
                writeJSON(w, http.StatusInternalServerError, APIError{Detail: "internal server error"})
            }
        }
    }
}

func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}

// The handler itself is now clean — it just returns an error.
func getUserHandler(w http.ResponseWriter, r *http.Request) error {
    id := r.URL.Query().Get("id")
    if id == "" {
        return APIError{Status: http.StatusBadRequest, Detail: "missing user ID"}
    }
    if id != "42" {
        return ErrNotFound
    }
    return writeJSON(w, http.StatusOK, map[string]string{"name": "Alice"})
}

func main() {
    http.HandleFunc("/user", MakeHTTPHandler(getUserHandler))
    fmt.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The handler is now a clean function that **returns a value** — no response logic, no formatting, no boilerplate. All of that lives in one place: `MakeHTTPHandler`.

---

## Applying the Same Pattern to Gin

The same approach works with Gin. Define a custom handler type and a `MakeHandler` wrapper:

```go
package main

import (
    "errors"
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
)

var (
    ErrNotFound     = errors.New("resource not found")
    ErrUnauthorized = errors.New("unauthorized access")
)

type APIError struct {
    Status int    `json:"-"`
    Detail string `json:"error"`
}

func (e APIError) Error() string { return e.Detail }

type APIHandler func(c *gin.Context) error

func MakeHandler(h APIHandler) gin.HandlerFunc {
    return func(c *gin.Context) {
        if err := h(c); err != nil {
            var apiErr APIError
            switch {
            case errors.As(err, &apiErr):
                c.JSON(apiErr.Status, apiErr)
            case errors.Is(err, ErrNotFound):
                c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
            case errors.Is(err, ErrUnauthorized):
                c.JSON(http.StatusUnauthorized, gin.H{"error": err.Error()})
            default:
                fmt.Printf("[Internal Error]: %v\n", err)
                c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            }
        }
    }
}

func getUserHandler(c *gin.Context) error {
    id := c.Query("id")
    if id == "" {
        return APIError{Status: http.StatusBadRequest, Detail: "missing user ID"}
    }
    if id != "42" {
        return ErrNotFound
    }
    c.JSON(http.StatusOK, gin.H{"name": "Alice"})
    return nil
}

func main() {
    r := gin.Default()
    r.GET("/user", MakeHandler(getUserHandler))
    r.Run(":8080")
}
```

### Prefer Gin's Native `c.Error()`?

If you'd rather not define a custom handler type and prefer staying idiomatic to Gin, you can use `c.Error()` to attach errors to the context and handle them in a middleware:

```go
func ErrorMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) == 0 {
            return
        }

        err := c.Errors.Last().Err
        var apiErr APIError

        switch {
        case errors.As(err, &apiErr):
            c.JSON(apiErr.Status, apiErr)
        case errors.Is(err, ErrNotFound):
            c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        default:
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        }
    }
}

func getUserHandler(c *gin.Context) {
    id := c.Query("id")
    if id == "" {
        c.Error(APIError{Status: http.StatusBadRequest, Detail: "missing user ID"})
        return
    }
    if id != "42" {
        c.Error(ErrNotFound)
        return
    }
    c.JSON(http.StatusOK, gin.H{"name": "Alice"})
}

func main() {
    r := gin.New()
    r.Use(ErrorMiddleware())
    r.GET("/user", getUserHandler)
    r.Run(":8080")
}
```

!!! note "Which Approach Should You Use?"
    Both are valid. The key difference:

    | | Custom `APIHandler` wrapper | Native `c.Error()` middleware |
    |---|---|---|
    | **Compile-time safety** | ✅ Enforced by the type system | ❌ Handler signature unchanged |
    | **Idiomatic to Gin** | Somewhat custom | ✅ Native Gin pattern |
    | **Centralized error logic** | ✅ In `MakeHandler` | ✅ In middleware |
    | **Easy to test** | ✅ Return values are explicit | Requires inspecting `c.Errors` |

---

## Why This Matters

!!! success "The Core Benefit"
    Making handlers return errors **separates concerns**. Your handlers describe *what went wrong*. The wrapper or middleware decides *how to respond*. These are two different responsibilities — keeping them apart makes both easier to read, test, and change.

Concretely:

- **No duplication.** Error formatting lives in one place, not scattered across dozens of handlers.
- **Consistent responses.** Every error path goes through the same code, so clients always get the same response shape.
- **Easier testing.** A handler that returns a value is trivial to unit test. You don't need to inspect `http.ResponseWriter` to know what happened.
- **Safer defaults.** Unhandled errors fall through to a single `default` case that logs internally and never leaks stack traces to clients.

---

## Conclusion

The standard `http.HandlerFunc` signature was designed for flexibility, not ergonomics. Returning errors from handlers is a small, low-cost change to your type signatures that pays dividends across your entire codebase.

Give it a try in your next service — and let me know what you think in the comments.