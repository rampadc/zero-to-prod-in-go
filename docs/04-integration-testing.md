# Integration testing

Let's recap. This is the testing code we have. It works, but it's not actually doing an integration test. It's not spinning up a webserver. It mocks one out with `httptest`.

```go
// /test/api-server/health_check_test.go
package test

import (
	apiserver "github.com/rampadc/zero-to-prod-in-go/internal/api-server"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestHealthCheck(t *testing.T) {
	router := apiserver.SetupRouter()

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/health_check", nil)
	router.ServeHTTP(w, req)

	assert.Equal(t, 200, w.Code)
}
```

Our goal is to do a full integration test with an HTTP client hitting a webserver running our Gin router. To accomplish this, we would:

1. Spawn a web server with a random port. 
2. Send requests against this new web server. 
3. Assert.

This requires having 2 processes: 1 for the webserver and 1 for running queries against the webserver. Sounds like a need for parallelism or at least concurrency has arisen.

We need to have a brief look at goroutines.

## Concurrency in Go

Go provides two concurrency primitives: goroutines and channels. 

Any functions in Go prepended with a `go` keyword will run in a concurrent process called a *goroutine*. The caller can continue uninterrupted rather than having to wait for the *goroutine* to return. Goroutines that can't terminate (stuck in infinite loop or blocked) will exist for the lifetime of the application. If there are lots of these, this leads to a *goroutine leak*.

Channels is a primitive that allows communication between goroutines. Each channel can transmit and receive values of a single type.

```go
ch <- value // sending value to a channel
sink = <-ch // receiving a value from a channel, and assigning to sink
<-ch // receiving from a channel, and discarding the result
```

More details on Go concurrency can be found on the [Tour of Go](https://go.dev/tour/concurrency).

## Testing out the new testing strategy

### Code change

Let's move our server code into a goroutine. I'll make a new file call `health_check_2_test.go` for this.

```go
// /test/health_check_2_test.go
package test

import (
	"fmt"
	apiserver "github.com/rampadc/zero-to-prod-in-go/internal/api-server"
	"github.com/stretchr/testify/assert"
	"net/http"
	"testing"
)

func startTestServer() {
    // This is basically our main.go code
	r := apiserver.SetupRouter()
	err := r.Run(":8080")
	if err != nil {
		return
	}
}

func TestHealthCheck2(t *testing.T) {
	go startTestServer()

	resp, err := http.Get(fmt.Sprintf("http://localhost:8080/health_check"))
	if err != nil {
		t.Fatal(err)
	}

	assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

Let's check it out.

```sh
go test test/api-server/health_check_2_test.go
```

```sh
ok      command-line-arguments  0.457s
```

That works! That was straightforward. `startTestServer()` contains the same code as in our `main.go`. In `TestHealthCheck2`, we're starting the server in a goroutine, then issue a HTTP GET request against the webserver.

### Randomise the port number

Next up, we need to make sure the test server starts on a random port so it doesn't conflict with our app, and maybe with other tests when we parallelise them. To get a random port, we can pass in `":0"` into the address portion of `Run()`. However, to run queries against that address, we need to know which port the server has chosen.

Looking at the signature for `Run()`:

```go
// Run attaches the router to a http.Server and starts listening and serving HTTP requests.
// It is a shortcut for http.ListenAndServe(addr, router)
// Note: this method will block the calling goroutine indefinitely unless an error happens.
func (engine *Engine) Run(addr ...string) (err error)
```

It does not return an `int` for the port.

Following the thread to `http.ListenAndServe(addr, router)`:

```go
// ListenAndServe listens on the TCP network address addr and then calls
// [Serve] with handler to handle requests on incoming connections.
// Accepted connections are configured to enable TCP keep-alives.
//
// The handler is typically nil, in which case [DefaultServeMux] is used.
//
// ListenAndServe always returns a non-nil error.
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```

Same problem. Here, however, inspecting the code for `ListenAndServe`, there's a little nugget!

```go
ln, err := net.Listen("tcp", addr)
```

`net.Listen()` returns a listener and an error. The `Listener` is a generic network listener which provides an `Addr() Addr` function that returns the network address, i.e., we can extract the port number from this.

Let's change our `startTestServer()` function.

```go
// Changing the signature of this function to return a port integer and the error
func startTestServer() (int, error) {
	router := apiserver.SetupRouter()

    // Let the operating system choose a random port for us with ":0"
	listener, err := net.Listen("tcp", ":0")
	if err != nil {
		return 0, err
	}

    // Extract the Port from listener
    // (*net.TCPAddr) converts the net.Addr interface to a *net.TCPAddr pointer
    // This is necessary because we want to access the Port field, which is
    // specific to TCP addresses
	port := listener.Addr().(*net.TCPAddr).Port
	err = http.Serve(listener, router)
	if err != nil {
		return 0, err
	}

    // return the port
	return port, nil
}
```

For the test to get the port number, we need to return it. Additionally, I want to fail the test if the server code has any errors.

But now we have a problem. Because startTestServer() is being run as a goroutine. It doesn't return within the testing function. There's no way we can get the port number. To solve this, we can either use a channel to pass the port number around, *or* we can put only the server creation part in a goroutine.

```go
func startTestServer() (int, error) {
	router := apiserver.SetupRouter()

	listener, err := net.Listen("tcp", ":0")
	if err != nil {
		return 0, err
	}

	port := listener.Addr().(*net.TCPAddr).Port

    // wrap the server starting part in a goroutine.
    // this is an anonymous go function. Inside, the anonymous go function
    // can access outer variables
	go func() {
		err = http.Serve(listener, router)
		if err != nil {
			panic(err)
		}
	}()

	return port, nil
}

func TestHealthCheck2(t *testing.T) {
    // startTestServer() is no longer being used as a goroutine
    // so that we can get the port number
	port, err := startTestServer()
	if err != nil {
		t.Fatal(err)
	}

    // port number being used in HTTP GET
	resp, err := http.Get(fmt.Sprintf("http://localhost:%d/health_check", port))
	if err != nil {
		t.Fatal(err)
	}

	assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

Rerunning the test

```
go test test/api-server/health_check_2_test.go
```

```
ok      command-line-arguments  0.230s
```

Perfect.

Let's do some light refactoring. Copy the code from `health_check_2_test.go` to `health_check_test.go` and delete the `_2`.

Then we can move the code for `startTestServer()` into `/internal/api-server/router.go` as well, renaming it to `StartTestServer()` for public access. This way, we can reuse this in other tests.

The code should now look like this

```sh
.
├── go.mod
├── go.sum
├── internal
│   └── api-server
│       └── router.go
├── main.go
└── test
    └── api-server
        └── health_check_test.go
```

```go
// /internal/api-server/router.go
package apiserver

import (
	"fmt"
	"net"
	"net/http"

	"github.com/gin-gonic/gin"
)

func SetupRouter() *gin.Engine {
	// Disable Console Color
	// gin.DisableConsoleColor()
	r := gin.Default()

	r.GET("/:name", func(c *gin.Context) {
		name := c.Params.ByName("name")
		// c.Data(http.StatusOK, "text/plain", []byte(fmt.Sprintf("Hello %s", name)))
		c.String(http.StatusOK, fmt.Sprintf("Hello %s", name))
	})

	r.GET("/health_check", func(c *gin.Context) {
		fmt.Printf("Health check got pinged\n")
		c.Status(http.StatusOK)
	})

	return r
}

func StartTestServer() (int, error) {
	router := SetupRouter()

	listener, err := net.Listen("tcp", ":0")
	if err != nil {
		return 0, err
	}

	port := listener.Addr().(*net.TCPAddr).Port
	go func() {
		err = http.Serve(listener, router)
		if err != nil {
			panic(err)
		}
	}()

	return port, nil
}
```

```go
// /test/api-server/health_check_test.go
package test

import (
	"fmt"
	apiserver "github.com/rampadc/zero-to-prod-in-go/internal/api-server"
	"github.com/stretchr/testify/assert"
	"net/http"
	"testing"
)

func TestHealthCheck(t *testing.T) {
	port, err := apiserver.StartTestServer()
	if err != nil {
		t.Fatal(err)
	}

	// port number being used in HTTP GET
	resp, err := http.Get(fmt.Sprintf("http://localhost:%d/health_check", port))
	if err != nil {
		t.Fatal(err)
	}

	assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```
