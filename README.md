# treemux - fast and flexible HTTP router

[![Build Status](https://travis-ci.com/vmihailenco/treemux.png?branch=master)](https://travis-ci.com/vmihailenco/treemux)
[![PkgGoDev](https://pkg.go.dev/badge/github.com/vmihailenco/treemux)](https://pkg.go.dev/github.com/vmihailenco/treemux)
[![Chat](https://discordapp.com/api/guilds/752070105847955518/widget.png)](https://discord.gg/rWtp5Aj)

> :heart: [**Uptrace.dev** - distributed traces, logs, and errors in one place](https://uptrace.dev)

- [Basic example](/example/basic/)
- [CORS example](/example/cors/)
- [Debug logging](/extra/reqlog/)
- [Gzip compression](/extra/treemuxgzip/)
- [OpenTelemetry integration](/extra/treemuxotel/)
- [Writing REST API with Go and PostgreSQL](https://pg.uptrace.dev/rest-api/)
- [RealWorld example application](https://github.com/uptrace/go-treemux-realworld-example-app)
- [Reference](https://pkg.go.dev/github.com/vmihailenco/treemux)

High-speed, flexible, tree-based HTTP router for Go. It is like Julien Schmidt's httprouter, but
supports more flexible routing.

## Installing with Go Modules

When using Go Modules, import this repository with `import "github.com/vmihailenco/treemux"` to
ensure that you get the right version.

## Handler

The handler is a simple function with the prototype
`func(w http.ResponseWriter, req treemux.Request) error`. A `treemux.Request` contains route name
and parameters parsed from wildcards and catch-alls in the URL. This type is aliased as
`treemux.HandlerFunc`.

```go
import "github.com/vmihailenco/treemux"

router := treemux.New()

group := router.NewGroup("/api/v1")

group.GET("/:id", func(w http.ResponseWriter, req treemux.Request) error {
    id := req.Param("id")
    return treemux.JSON(w, treemux.H{
        "url": fmt.Sprintf("GET /api/v1/%s", id),
        "route": req.Route(),
    })
})

log.Println(http.ListenAndServe(":8080", router))
```

treemux supports centralized handling of errors returned by handlers:

```go
router := treemux.New(
    treemux.WithErrorHandler(errorHandler),
)

func errorHandler(w http.ResponseWriter, req treemux.Request, err error) {
    w.WriteHeader(500)
    _ = treemux.JSON(w, treemux.H{
        "message": "Internal Server Error",
    })
}
```

## Middlewares

Middleware is a function that wraps a handler with another function:

```go
func corsMiddleware(next treemux.HandlerFunc) treemux.HandlerFunc {
    return func(w http.ResponseWriter, req treemux.Request) error {
        if origin := req.Header.Get("Origin"); origin != "" {
            h := w.Header()
            h.Set("Access-Control-Allow-Origin", origin)
            h.Set("Access-Control-Allow-Credentials", "true")
        }
        return next(w, req)
    }
}

router = treemux.New(treemux.WithMiddleware(corsMiddleware))
```

## Routing Rules

The syntax here is modeled after httprouter. Each variable in a path may match on one segment only,
except for an optional catch-all variable at the end of the URL.

Some examples of valid URL patterns are:

- `/post/all`
- `/post/:postid`
- `/post/:postid/page/:page`
- `/post/:postid/:page`
- `/images/*path`
- `/favicon.ico`
- `/:year/:month/`
- `/:year/:month/:post`
- `/:page`

Note that all of the above URL patterns may exist concurrently in the router.

Path elements starting with `:` indicate a wildcard in the path. A wildcard will only match on a
single path segment. That is, the pattern `/post/:postid` will match on `/post/1` or `/post/1/`, but
not `/post/1/2`.

A path element starting with `*` is a catch-all, whose value will be a string containing all text in
the URL matched by the wildcards. For example, with a pattern of `/images/*path` and a requested URL
`images/abc/def`, path would contain `abc/def`. A catch-all path will not match an empty string, so
in this example a separate route would need to be installed if you also want to match `/images/`.

#### Using : and \* in routing patterns

The characters `:` and `*` can be used at the beginning of a path segment by escaping them with a
backslash. A double backslash at the beginning of a segment is interpreted as a single backslash.
These escapes are only checked at the very beginning of a path segment; they are not necessary or
processed elsewhere in a token.

```go
router.GET("/foo/\\*starToken", handler) // matches /foo/*starToken
router.GET("/foo/star*inTheMiddle", handler) // matches /foo/star*inTheMiddle
router.GET("/foo/starBackslash\\*", handler) // matches /foo/starBackslash\*
router.GET("/foo/\\\\*backslashWithStar") // matches /foo/\*backslashWithStar
```

### Routing Groups

Lets you create a new group of routes with a given path prefix. Makes it easier to create clusters
of paths like:

- `/api/v1/foo`
- `/api/v1/bar`

To use this you do:

```go
router = treemux.New()

api := router.NewGroup("/api/v1")
api.GET("/foo", fooHandler) // becomes /api/v1/foo
api.GET("/bar", barHandler) // becomes /api/v1/bar
```

Or using `WithGroup`:

```go
router.WithGroup("/api/v1", func(g *treemux.Group) {
    g.GET("/foo", fooHandler) // becomes /api/v1/foo
    g.GET("/bar", barHandler) // becomes /api/v1/bar
})
```

More complex example:

```go
router := treemux.New()

g := router.NewGroup("/api/v1", treemux.WithMiddleware(ipRateLimitMiddleware))

g.NewGroup("/users/:user_id",
    treemux.WithMiddleware(authMiddleware),
    treemux.WithGroup(func(g *treemux.Group) {
        g.GET("", userHandler)

        g = g.WithMiddleware(adminMiddleware)

        g.PUT("", updateUserHandler)
        g.DELETE("", deleteUserHandler)
    }))

g.NewGroup("/projects/:project_id/articles/:article_id",
    treemux.WithMiddleware(authMiddleware),
    treemux.WithMiddleware(projectMiddleware),
    treemux.WithGroup(func(g *treemux.Group) {
        g.GET("", articleHandler)

        g.Use(quotaMiddleware)

        g.POST("", createArticleHandler)
        g.PUT("", updateArticleHandler)
        g.DELETE("", deleteArticleHandler)
    }))
```

### Routing Priority

The priority rules in the router are simple.

1. Static path segments take the highest priority. If a segment and its subtree are able to match
   the URL, that match is returned.
2. Wildcards take second priority. For a particular wildcard to match, that wildcard and its subtree
   must match the URL.
3. Finally, a catch-all rule will match when the earlier path segments have matched, and none of the
   static or wildcard conditions have matched. Catch-all rules must be at the end of a pattern.

So with the following patterns adapted from [simpleblog](https://www.github.com/dimfeld/simpleblog),
we'll see certain matches:

```go
router = treemux.New()
router.GET("/:page", pageHandler)
router.GET("/:year/:month/:post", postHandler)
router.GET("/:year/:month", archiveHandler)
router.GET("/images/*path", staticHandler)
router.GET("/favicon.ico", staticHandler)
```

#### Example scenarios

- `/abc` will match `/:page`
- `/2014/05` will match `/:year/:month`
- `/2014/05/really-great-blog-post` will match `/:year/:month/:post`
- `/images/CoolImage.gif` will match `/images/*path`
- `/images/2014/05/MayImage.jpg` will also match `/images/*path`, with all the text after `/images`
  stored in the variable path.
- `/favicon.ico` will match `/favicon.ico`

### Special Method Behavior

If TreeMux.HeadCanUseGet is set to true, the router will call the GET handler for a pattern when a
HEAD request is processed, if no HEAD handler has been added for that pattern. This behavior is
enabled by default.

Go's http.ServeContent and related functions already handle the HEAD method correctly by sending
only the header, so in most cases your handlers will not need any special cases for it.

By default TreeMux.OptionsHandler is a null handler that doesn't affect your routing. If you set the
handler, it will be called on OPTIONS requests to a path already registered by another method. If
you set a path specific handler by using `router.OPTIONS`, it will override the global Options
Handler for that path.

### Trailing Slashes

The router has special handling for paths with trailing slashes. If a pattern is added to the router
with a trailing slash, any matches on that pattern without a trailing slash will be redirected to
the version with the slash. If a pattern does not have a trailing slash, matches on that pattern
with a trailing slash will be redirected to the version without.

The trailing slash flag is only stored once for a pattern. That is, if a pattern is added for a
method with a trailing slash, all other methods for that pattern will also be considered to have a
trailing slash, regardless of whether or not it is specified for those methods too. However this
behavior can be turned off by setting TreeMux.RedirectTrailingSlash to false. By default it is set
to true.

One exception to this rule is catch-all patterns. By default, trailing slash redirection is disabled
on catch-all patterns, since the structure of the entire URL and the desired patterns can not be
predicted. If trailing slash removal is desired on catch-all patterns, set
TreeMux.RemoveCatchAllTrailingSlash to true.

```go
router = treemux.New()
router.GET("/about", pageHandler)
router.GET("/posts/", postIndexHandler)
router.POST("/posts", postFormHandler)

GET /about will match normally.
GET /about/ will redirect to /about.
GET /posts will redirect to /posts/.
GET /posts/ will match normally.
POST /posts will redirect to /posts/, because the GET method used a trailing slash.
```

### Custom Redirects

RedirectBehavior sets the behavior when the router redirects the request to the canonical version of
the requested URL using RedirectTrailingSlash or RedirectClean. The default behavior is to return a
301 status, redirecting the browser to the version of the URL that matches the given pattern.

These are the values accepted for RedirectBehavior. You may also add these values to the
RedirectMethodBehavior map to define custom per-method redirect behavior.

- Redirect301 - HTTP 301 Moved Permanently; this is the default.
- Redirect307 - HTTP/1.1 Temporary Redirect
- Redirect308 - RFC7538 Permanent Redirect
- UseHandler - Don't redirect to the canonical path. Just call the handler instead.

#### Rationale/Usage

On a POST request, most browsers that receive a 301 will submit a GET request to the redirected URL,
meaning that any data will likely be lost. If you want to handle and avoid this behavior, you may
use Redirect307, which causes most browsers to resubmit the request using the original method and
request body.

Since 307 is supposed to be a temporary redirect, the new 308 status code has been proposed, which
is treated the same, except it indicates correctly that the redirection is permanent. The big caveat
here is that the RFC is relatively recent, and older or non-compliant browsers will not handle it.
Therefore its use is not recommended unless you really know what you're doing.

Finally, the UseHandler value will simply call the handler function for the pattern, without
redirecting to the canonical version of the URL.

### RequestURI vs. URL.Path

#### Escaped Slashes

Go automatically processes escaped characters in a URL, converting + to a space and %XX to the
corresponding character. This can present issues when the URL contains a %2f, which is unescaped to
'/'. This isn't an issue for most applications, but it will prevent the router from correctly
matching paths and wildcards.

For example, the pattern `/post/:post` would not match on `/post/abc%2fdef`, which is unescaped to
`/post/abc/def`. The desired behavior is that it matches, and the `post` wildcard is set to
`abc/def`.

Therefore, this router defaults to using the raw URL, stored in the Request.RequestURI variable.
Matching wildcards and catch-alls are then unescaped, to give the desired behavior.

TL;DR: If a requested URL contains a %2f, this router will still do the right thing. Some Go HTTP
routers may not due to [Go issue 3659](https://github.com/golang/go/issues/3659).

#### http Package Utility Functions

Although using RequestURI avoids the issue described above, certain utility functions such as
`http.StripPrefix` modify URL.Path, and expect that the underlying router is using that field to
make its decision. If you are using some of these functions, set the router's `PathSource` member to
`URLPath`. This will give up the proper handling of escaped slashes described above, while allowing
the router to work properly with these utility functions.

## Error Handlers

### ErrorHandler

To handle errors returned by handlers, use `TreeMux.ErrorHandler`:

```go
router.ErrorHandler = func(w http.ResponseWriter, req treemux.Request, err error) {
    w.WriteHeader(500)
    _, _ = w.Write([]byte("Internal Server Error"))
}
```

### NotFoundHandler

`TreeMux.NotFoundHandler` can be set to provide custom 404-error handling. The default
implementation is Go's `http.NotFound` function.

### MethodNotAllowedHandler

If a pattern matches, but the pattern does not have an associated handler for the requested method,
the router calls the MethodNotAllowedHandler. The default version of this handler just writes the
status code `http.StatusMethodNotAllowed`.

## Unexpected Differences from Other Routers

This router is intentionally light on features in the name of simplicity and performance. When
coming from another router that does heavier processing behind the scenes, you may encounter some
unexpected behavior. This list is by no means exhaustive, but covers some nonobvious cases that
users have encountered.

### httprouter and catch-all parameters

When using `httprouter`, a route with a catch-all parameter (e.g. `/images/*path`) will match on
URLs like `/images/` where the catch-all parameter is empty. This router does not match on empty
catch-all parameters, but the behavior can be duplicated by adding a route without the catch-all
(e.g. `/images/`).

## httptreemux

This is a fork of [httptreemux](https://github.com/dimfeld/httptreemux). Most of the code is written
by Daniel Imfeld.

### Changes from httptreemux

- Thin wrapper `treemux.Request` around `http.Request` to expose route via `Request.Route` and route
  params via `req.Params`.

- Setting a `context.Context` does not require an allocation.

- Centralized error handling. Each handler returns an error which is handled by global
  `WithErrorHandler` func.

- More efficient params encoding using a slice instead of a map.

- `Group` is immutable to avoid accidental leaking of middlewares into the group.
