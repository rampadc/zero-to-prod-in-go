# Testing the endpoint

Manual testing is time-consuming. We'd like to automate as much as possible. In future sections, you'll see how this automation extends to deploying to a Kubernetes cluster as well!

The API endpoint we expose is a contract between us and the client. Testing an API endpoint should be tested from the perspective of the client, i.e., the browser. To that, we're effectively doing black box testing.

## Where do I put my tests?

Ah, the age-old question of how to organise your Go project. There's the [official way](https://go.dev/doc/modules/layout) and the [unofficial way](https://github.com/golang-standards/project-layout). Both recommends that you start off simple: a single `main.go` and a `go.mod` files in your project. But what does a bigger project look like? The unofficial standard's got us covered.

```sh
.
├── LICENSE.md
├── Makefile
├── README.md
# OpenAPI, Swagger, JSON schema, protocol definition files
├── api
│   └── README.md
# Images, logos, etc. for your repository
├── assets
│   └── README.md
# Packaging and CI artifacts
├── build
│   ├── README.md
#### /build/ci: travis.yml, drone files
│   ├── ci
#### /build/package: Dockerfile, AMI, deb/rpm/pkg packages
│   └── package
# Main applications. The directory should match the name
# of the executable. It's common to have a `main` function
# that imports and invokes code from `/internal` and `/pkg`
# and nothing else.
├── cmd
│   ├── README.md
│   └── _your_app_
# Configuration file templates or default configs.
# `confd`, `consul-template` goes here
# Let's look at hierarchical configs later!
├── configs
│   └── README.md
# IaC, PaaS deployment (K8s, Helm, Terrraform)
├── deployments
│   └── README.md
# Design and user documentation, godoc
# Realistically, this guide should live in a `docs` folder
# in the zero-to-prod-in-go source code
├── docs
│   └── README.md
# Examples
├── examples
│   └── README.md
# Self-explanatory
├── githooks
│   └── README.md
├── go.mod
# System init and process manager/superivsor configs
├── init
│   └── README.md
# Private application and library code. This is code you don't want others
# importing into their applications or libraries.
├── internal
│   ├── README.md
#### our actual code can go into `/internal/app`
│   ├── app
#### and the shared code by our apps could go into `/internal/pkg`
│   └── pkg
# Library code that other applications can import
├── pkg
│   ├── README.md
│   └── _your_public_lib_
# Makefile goes in here. Scripts to perform build, test, install, etc.
├── scripts
│   └── README.md
# Self-explanatory
├── test
│ └── README.md
# External tools, forked code or 3rd party
├── third_party
│   └── README.md
# Supporting tools for this project, can import from /pkg and /internal
├── tools
│   └── README.md
# Application dependencies. `go mod vendor` command creates the `/vendor` directory
├── vendor
│   └── README.md
# SPA,s tatic web assets, server-side templates
├── web
│   ├── README.md
│   ├── app
│   ├── static
│   └── template
# Where your frontend source code lives
└── website
	└── README.md
```

## Restructuring the directories

Let's move things around. We're assuming our app is going to be *huge!* later. Maybe you're embedding this web server in a Kubernetes operator, or perhaps a command line interface where this is just another app. I'll call this Gin app the `api-server`.

The final folder structure looks something like this

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

First, I moved `main.go` into `/cmd/api-server`. Then, because we want to be able to setup and teardown the Gin engine at will, the code should live in another file: `/internal/api-server/router.go`. This will allow us to call `setupRouter()` in any other functions, say in `/test/api-server/health_check_test.go`.

The contents of the files look something like this:

```go
// /internal/api-server/router.go
package apiserver

import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
)

func SetupRouter() *gin.Engine {
	// Disable Console Color
	// gin.DisableConsoleColor()
	r := gin.Default()

	r.GET("/:name", func(c *gin.Context) {
		name := c.Params.ByName("name")
		c.String(http.StatusOK, fmt.Sprintf("Hello %s", name))
	})

	r.GET("/health_check", func(c *gin.Context) {
		c.Status(http.StatusOK)
	})

	return r
}

```

```go
// main.go
package main

import apiserver "github.com/rampadc/zero-to-prod-in-go/internal/api-server"

func main() {
	r := apiserver.SetupRouter()
	err := r.Run(":8080")
	if err != nil {
		return
	}
}
```

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

That's a lot of code change. Let's slow down a bit and talk about what's going on.

### Where did that massive github.com/rampadc/... come from?

To comply with conventions, I renamed my root Go module's name to `github.com/rampadc/zero-to-prod-in-go`.

The top part of my `go.mod` looks like this:

```
// go.mod
module github.com/rampadc/zero-to-prod-in-go

go 1.23.2

require github.com/gin-gonic/gin v1.10.0

require (
	// ...
```

The equivalent concept in Rust would be the keyword `crate`, denoting the root module.

### Simplifying commands

Moving things around sure makes things less cluttered, but now we have to type a lot more to do simple things, like

```sh
# Run the main app
go run main.go
# Run test
go run test/api-server/health_check_test.go
```

Well, for the main app, we can just run

```
go run .
```

As for tests, for now, you can specify a glob

```sh
go test test/**/*.go
```


### Why is setupRouter() now SetupRouter()?

Next, I moved the `setupRouter()` code into `/internal/api-server/router.go`. All functions in Go, by default, are private to that file or package.

To allow other files in my app to use the function, I need to capitalise the function name, thereby making it public.

In `main.go` and `health_check_test.go`, I can then import `SetupRouter()` by writing 

```go
// /test/api-server/health_check_test.go

import apiserver "github.com/rampadc/zero-to-prod-in-go/internal/api-server"
```

`apiserver` is an alias. It can be anything.

### Run the test

Let's run a test.

```sh
go run tests/api-server/health_check_test.go
```

```sh
ok      command-line-arguments  0.408s
```

It works! Let's change the assertion to see a failure.

```go
// /test/api-server/health_check_test.go
assert.Equal(t, 200, w.Code)
```

```sh
go test test/api-server/health_check_test.go
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /:name                    --> github.com/rampadc/zero-to-prod-in-go/internal/api-server.SetupRouter.func1 (3 handlers)
[GIN-debug] GET    /health_check             --> github.com/rampadc/zero-to-prod-in-go/internal/api-server.SetupRouter.func2 (3 handlers)
[GIN] 2024/11/12 - 17:44:53 | 200 |         500ns |       192.0.2.1 | GET      "/health_check"
--- FAIL: TestHealthCheck (0.00s)
    health_check_test.go:18: 
                Error Trace:    /Users/cong/Dev/z2p-in-go/test/api-server/health_check_test.go:18
                Error:          Not equal: 
                                expected: 201
                                actual  : 200
                Test:           TestHealthCheck
FAIL
FAIL    command-line-arguments  0.285s
FAIL
```

Yay! That's what we want to see `expected` != `actual`. I'm going to change the expected value back to `200` for the test to pass.

## Are we actually doing integration testing?

Yes, our tests run, but technically, we're not doing integration testing. We're not hitting a live endpoint, i.e., seeing it from the perspective of an API caller.

In the test, `ServeHTTP()` itself does not start a webserver. It's a mock HTTP endpoint. This method comes from the [`httptest` package](https://pkg.go.dev/net/http/httptest).

Let's break it down line by line.

```go
// w implements the `ResponseWriter` interface. It's used to record all responses from the handler
w := httptest.NewRecorder()

// This line creates a mock request to the server.
req := httptest.NewRequest("GET", "/health_check", nil)

// Delegate requests to the Gin router
router.ServeHTTP(w, req)
```

Using httptest approach lets our tests run faster as they don't have to start and stop a web server. I mean... if works and it's common practice to do endpoint testing within the Go community, then this is the way.

But, let's spin up a new HTTP server for testing anyway, and see how we fare.
