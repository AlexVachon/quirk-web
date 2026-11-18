# web

Build web apps and HTTP APIs in Quirk, Express-style: routes with
`:params`, middleware, JSON / forms / cookies, static files, page
templates, a keep-alive server, and a test client.

```quirk
use web

app := web.App()
app.use(web.logger())
app.use(web.static("public"))

app.get("/", fn(req, res) {
    res.render("index.html", {"title": "Home"})
})

app.get("/users/:id", fn(req, res) {
    res.json({"id": req.params["id"]})
})

app.post("/users", fn(req, res) {
    user: Map = req.json()
    res.status(201).json(user)
})

app.listen(3000)          // http://localhost:3000
```

Start a project with `quirk new --web my-site`: routes, a template, a
stylesheet, a JSON API and tests, ready to `quirk run src/index.qk`.

## Routes

`app.get / post / put / patch / delete / options / all(pattern, handler)`,
or `app.route(method, pattern, handler)`. A handler is `fn(req, res)`.

| Pattern | Matches | `req.params` |
|---|---|---|
| `/users` | `/users`, `/users/` | |
| `/users/:id` | `/users/42` | `{"id": "42"}` |
| `/users/:id/posts/:post` | `/users/7/posts/9` | `{"id": "7", "post": "9"}` |
| `/files/*path` | `/files/a/b.txt` | `{"path": "a/b.txt"}` |

Routes are tried in the order added. A path that matches only for other
methods answers 405 with an `Allow` header; HEAD runs the GET route.

Group routes in a `web.Router()` and mount it under a prefix:

```quirk
api := web.Router()
api.get("/items", list_items)
app.mount("/api", api)        // GET /api/items
```

## Requests

| | |
|---|---|
| `req.method`, `req.path`, `req.url` | `"GET"`, `"/users/42"` (decoded), `"/users/42?tab=1"` |
| `req.query` | Map of the query string |
| `req.params` | Map of the route's `:name` / `*name` parts |
| `req.header(name)` | a header ("" if absent; any case) — all in `req.headers` |
| `req.json()` | the body as JSON (HttpError 400 if it isn't) |
| `req.form()` | an urlencoded form body as a Map |
| `req.cookie(name)`, `req.cookies()` | cookies |
| `req.body` | the raw body — bytes as sent (uploads, binary) |
| `req.is_type("json")`, `req.accepts("html")` | Content-Type / Accept checks |
| `req.locals` | a Map middleware fills for later handlers |

## Responses

The first send wins. Chain `status` / `set` before it.

| | |
|---|---|
| `res.text(s)`, `res.html(s or html.Element)`, `res.json(value)`, `res.send(s)` | send a body |
| `res.status(201)` | the status code |
| `res.set(name, value)`, `res.get(name)` | headers (`append_header` for repeats) |
| `res.redirect("/login")` | 302 (or `code=301`, `303`, `307`, `308`) |
| `res.cookie(name, value, max_age=, http_only=, secure=, same_site=)`, `res.clear_cookie(name)` | cookies |
| `res.file(path, download=false)` | a file, with its type and Last-Modified (304 when unchanged) |
| `res.render("page.html", data)` | a template from `app.views` (default `views/`) |
| `res.end()`, `res.send_status(404)` | no body / just a status |

## Middleware

`app.use(fn(req, res, next) { ... })` runs before the routes, in order; call
`next()` to go on, or answer to stop there. `path=` limits it:

```quirk
app.use(fn(req, res, next) {
    if req.cookie("session") == "" {
        res.redirect("/login")
        return
    }
    next()
}, path="/admin")
```

Included: `web.logger()` (one line per request), `web.static(dir, prefix="/")`
(files, index.html for folders, no way out of `dir`), `web.cors(origin="*")`.

## Errors

Throw `web.HttpError(404, "no such user")` from anywhere in a handler to
answer with that status and message. Anything else thrown is a 500 (the
error is printed, not sent). Replace either:

```quirk
app.on_error(fn(err, req, res) { res.status(500).json({"error": err.message}) })
app.on_not_found(fn(req, res) { res.status(404).render("404.html") })
```

## Sessions and login

```quirk
app.use(web.sessions())

app.post("/login", fn(req, res) {
    f := req.form()
    if not check_password(f["name"], f["password"]) { throw web.HttpError(403, "wrong password") }
    web.login(req, f["name"])                 // a new session id, then the user stored
    res.redirect("/account")
})
app.post("/logout", fn(req, res) {
    web.logout(req)                           // the session ends, the cookie is cleared
    res.redirect("/")
})

app.use(web.require_login(), path="/account") // logged-in visitors only
app.get("/account", fn(req, res) { res.text("Hello " + String(req.user())) })
```

- `req.session()` is the visitor's session: `get(key, default)`, `set`,
  `has`, `remove`, `clear`, `regenerate()`, `destroy()`. Its values stay on
  the server; the browser holds only a random id, signed, in an HttpOnly,
  SameSite=Lax cookie — sent only once the session has something in it.
- `web.login(req, user)` stores any value (a name, a Map) as `req.user()`,
  after giving the session a new id (an id seen before logging in can't be
  used to act as the user).
- `web.require_login(redirect="/login")` sends a page request to
  `/login?next=<the page>`, and answers anything else (an API call, a form
  post) with 401.
- `web.sessions(secret=…, minutes=10080, cookie="quirk_session",
  secure=false)`: sessions live in the server's memory and end when it
  restarts, or `minutes` after the visitor's last request. `secret` signs
  the cookie (random per start by default; set one to keep cookies valid
  across restarts). `secure=true` behind HTTPS.
- `web.TestClient` keeps cookies, so tests can log in.

## Templates

`views/page.html`:

```html
<h1>{{ title }}</h1>                         <!-- HTML-escaped -->
{% if user.admin %}<b>admin</b>{% elif user %}hi {{ user.name }}{% else %}guest{% end %}
<ul>
{% for item in items %}<li>{{ loop.index }}. {{ item.name | upper }}</li>{% else %}<li>none</li>{% end %}
</ul>
{{{ raw_html }}}  {{ intro | raw }}  {{ items | length }}  {{ data | json }}
{% include "footer.html" %}
{# a comment #}
```

Values come from the data Map: dotted names reach into Maps and Lists
(`user.name`, `items.0`, `items.length`); `not`, `==`, `!=` in conditions.
`web.Template(text).render(data)` and `web.render_file(path, data)` work
outside a response; files are cached until they change.

## Testing

`web.TestClient(app)` calls the app without a network, keeping cookies
like a browser:

```quirk
define test_home() -> void {
    c := web.TestClient(make_app())
    res := c.get("/")
    test.assert_eq(res.status_code, 200)
    created := c.post_json("/api/users", {"name": "Ada"})
    test.assert_eq(created.status_code, 201)
}
```

`get`, `post`, `put`, `delete`, `post_json`, `post_form`, `request(method, path, body, headers)`.

## The server

`app.listen(port=3000, host="127.0.0.1")` serves until `app.stop()`:
HTTP/1.1 with keep-alive, one task per connection, bodies by
Content-Length or chunked, `app.max_body` (10 MB) and an idle timeout
(`app.timeout_ms`, 15 s). Use `host="0.0.0.0"` to accept connections from
other machines. HTTPS, HTTP/2, WebSockets and multipart uploads aren't
built in yet — put a reverse proxy (Caddy, nginx) in front for TLS.

## Requirements

Quirk 7.42.0 or newer (binary-safe sockets and files, the byte helpers
in the runtime), and the `crypto` package (bundled with Quirk).

## License

MIT
