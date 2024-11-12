# Build a simple API endpoint

The first thing I'd like to accomplish is to get a a simple `/health_check` endpoint working. To do that, we'll start off with choosing a web framework to use. Gin seems to be the most mature with the greatest number of middleware: things we do not want to write.

## Wiring up Gin

Our starting point will be a *Hello World!* application with Gin.

```sh
mkdir z2p-in-go
cd z2p-in-go

go mod init z2p-in-go
go get -u github.com/gin-gonic/gin
```

Create a new file `main.go` and place this code in.

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func setupRouter() *gin.Engine {
	// Disable Console Color
	// gin.DisableConsoleColor()
	r := gin.Default()

	r.GET("/:name", func(c *gin.Context) {
		name := c.Params.ByName("name")
		c.Data(http.StatusOK, "text/plain", []byte("Hello "+name))
	})

	return r
}

func main() {
	r := setupRouter()
	// Listen and Server in 0.0.0.0:8080
	r.Run(":8080")
}
```

Run it with `go run main.go`.

```
curl http://localhost:8080/world
```

```
Hello world
```

Awesome. It works! In the terminal, you get a bit of a printout. But that actually looks pretty good! It's got logging and routing, a debug/release mode.

```sh
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /:name                    --> main.setupRouter.func1 (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on :8080
[GIN] 2024/11/12 - 00:15:26 | 404 |       1.042µs |             ::1 | GET      "/"
[GIN] 2024/11/12 - 00:15:27 | 200 |       14.75µs |             ::1 | GET      "/cong"
```

### Anatomy of a Gin application

Firstly, some core concepts about Gin.

* `gin.Engine` is the core component of the Gin framework. It's essentially a router that maps incoming HTTP requests to specific handler functions. In our case, the handler functions are anonymous functions with a specific signature `func(c *gin.Context)`.
* `gin.Default()` creates a new instance of `gin.Engine` with some default configurations like logging and middleware. It's a convenient way to get started quickly.

Now, let's walk through the code.

At the top, we have imports, specifically `net/http` and `github.com/gin-gonic/gin`. 

* `net/http` is a standard library in Go, and it provides functionalities for working with HTTP requests and responses. Here, we're using `http.StatusOK` which requires `net/http`.
* `github.com/gin-gonic/gin`: this imports Gin.

The `setupRouter` function is responsible for setting up the Gin router. We assign `gin.Default()` to `r` through `r := gin.Default()` to create a new router instance.

#### Endpoint - `r.`

The `r` variable contains the router instance. It lets us defines routes by specifiy the HTTP verb we want to use.

In our snippet, we have 

```go
r.GET("/:name", func(c *gin.Context) {
    name := c.Params.ByName("name")
    c.Data(http.StatusOK, "text/plain", []byte("Hello "+name))
})
```

This defines a route that handles GET requests to any path starting with `/`. The `:name` part is a placeholder for dynamic value captured in the URL. The function passed as the second argument is the handler function that will be called when a matching request arrives.

`c *gin.Context` represents the context of the current HTTP request. It provides access to various details like the request parameters, headers and methods for responding.

Using the `gin.Context`, we get the `name` by querying the URL params. `c.Data(http.StatusOK, "text/plain", []byte("Hello "+name))` sends a HTTP 200 OK response with the content "Hello " followed by the captured name.

`c.Data` is a method in the Gin context that allows you to send a raw response with a specific content and data. Here, we set the response code to be HTTP 200 OK and the response's `content-type` to be `text/plain`. `c.Data` expects a response body of a slice of bytes, which we achieved by using (`[]byte`).

:::tip

While `c.Data` is a versatile option that communicates our intent clearly, Gin also offers `c.String` as a shortcut to send plain text response with the formatted string. It automatically handles the conversion to `[]byte` internally.

```go
c.String(http.StatusOK, "Hello "+name)
```
:::

The default Gin example app returns a JSON object, but I opted for a plain text to keep up with the Hello world tradition.

Finally `return r` returns the built-out router instance.

In the `main` function, we now assign the router to a new variable with `r := setupRouter`, and start the HTTP server with `r.Run(":8080")`, listening at port 8080.

#### Why `setupRouter` returns a pointer?

In Go, pointers stores the memory address of another variable. `setupRouter()` returns a pointer to `gin.Engine` allows us to create only 1 engine and keep modifying that instead of creating many.

## Implementing the health check

For this endpoint, I just want to return a status 200. I was going to say "browsing through the docs, I found this", but that wasn't the case. Gin doesn't seem to have a reference manual like doc.rs with all of the available functions to be seen.

But that's okay. We can peruse the `gin.Context` file for hints, after understanding it provides us with the methods for responding. And I found what I needed `c.Status`.

```go
r.GET("/health_check", func(c *gin.Context) {
    c.Status(http.StatusOK)
})
```

Let's have a little test!

```sh
curl -v http://localhost:8080/health_check
* Host localhost:8080 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:8080...
* Connected to localhost (::1) port 8080
> GET /health_check HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Mon, 11 Nov 2024 22:51:46 GMT
< Content-Length: 0
< 
* Connection #0 to host localhost left intact
```

I gotta say, it's really simple to write an API endpoint in Gin!
