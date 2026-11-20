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
- `web.sessions(store=…, secret=…, minutes=10080, cookie="quirk_session",
  secure=false)`: sessions end `minutes` after the visitor's last request.
  `secure=true` behind HTTPS (in production).
- `web.TestClient` keeps cookies, so tests can log in.

### Sessions that last across restarts

By default sessions live in the server's memory: a restart — a deploy, or
every save under `quirk run --watch` — logs everyone out. Keep them in SQLite
instead:

```quirk
app.use(web.sessions(store=web.sqlite_store("sessions.db")))   // a file of their own
app.use(web.sessions(store=web.sqlite_store(db=db())))          // or the app's database
```

- They're in a `quirk_sessions` table (plus `quirk_session_meta`), made if it
  isn't there; the app's own `db.migrate` version isn't touched.
- Their values are kept as JSON: Strings, numbers, Bools, null, Lists and
  Maps (to keep a struct, keep its fields).
- A request writes only when the session's values changed, or once a minute
  to note it's still in use. Expired sessions are swept.
- A store of your own: a struct extending `web.SessionStore` (`load`, `save`,
  `remove`, `sweep`, `count`, `secret`).

**The secret** that signs the cookie, in this order: `secret=` if given; else
the `QUIRK_SESSION_SECRET` environment variable; else one the SQLite store
makes once and keeps in its database (cookies stay valid across restarts with
nothing to set up); else, for memory sessions, a new random one each start. Set
`QUIRK_SESSION_SECRET` (or `secret=`) when several servers share the sessions.

## Event streams

Keep a response open and send events as they happen — Server-Sent
Events, what the browser's `EventSource` reads:

```quirk
app.get("/clock", fn(req, res) {
    stream := res.event_stream()
    while stream.open {                       // false once the visitor is gone
        stream.send("time", String(sys.now_ms()))
        sys.sleep(1000)
    }
})
```

`stream.send(event, data, id="")` (multi-line data is fine), `stream.ping()`
(a comment line: keeps proxies awake and notices a closed tab sooner),
`stream.close()`. Each connection runs in its own task, so a stream doesn't
hold up other requests. In a `TestClient`, what was sent is in
`res._stream.sent`.

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

## Deploying

```sh
quirk bundle main.qk -o dist --docker     # dist/: the app, its runtime, its files, a Dockerfile
cd dist && ./main                         # runs there with no Quirk installed
```

`quirk bundle` builds the app to a native binary with the runtime next to it,
and copies what it reads from its folder (`index.html`, `public/`, `views/`,
templates) — never `.db` files. Run it from that folder. In production:

- **The address.** `app.listen` serves `127.0.0.1` (this machine only) unless
  `host=` says otherwise or `QUIRK_HOST` is set — `QUIRK_HOST=0.0.0.0` in a
  container (the generated Dockerfile sets it), or put a reverse proxy (Caddy,
  nginx) in front.
- **HTTPS.** Behind a proxy that does TLS, give sessions `secure=true`, so the
  cookie is never sent over plain HTTP.
- **The session secret.** Set `QUIRK_SESSION_SECRET` (a long random string:
  `openssl rand -hex 32`) — required when several servers share the sessions;
  with `web.sqlite_store` and one server, the store's own secret is enough.
- **Data.** A SQLite database the app writes lives in its folder: on a server,
  keep it out of the bundle (it is) and back it up; in a container, put it on a
  volume.

## Requirements

Quirk 7.56.0 or newer (`quirk bundle`, to deploy: 7.58.0), and the `crypto`
and `sqlite` packages (bundled with Quirk).

## License

MIT
