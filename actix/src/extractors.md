# Type-safe information extraction

Actix Web provides a facility for type-safe request information access called _extractors_ (i.e., `impl FromRequest`). There are lots of built-in extractor implementations (see [implementors](https://docs.rs/actix-web/latest/actix_web/trait.FromRequest.html#implementors)).

An extractor can be accessed as an argument to a handler function. Actix Web supports up to 12 extractors per handler function. Argument position does not matter.

```rust
async fn index(path: web::Path<(String, String)>, json: web::Json<MyInfo>) -> impl Responder {
    let path = path.into_inner();
    format!("{} {} {} {}", path.0, path.1, json.id, json.username)
}
```

## Path

[_Path_][pathstruct] provides information that is extracted from the request's path. Parts of the path that are extractable are called "dynamic segments" and are marked with curly braces. You can deserialize any variable segment from the path.

For instance, for resource that registered for the `/users/{user_id}/{friend}` path, two segments could be deserialized, `user_id` and `friend`. These segments could be extracted as a tuple in the order they are declared (e.g., `Path<(u32, String)>`).

```rust
use actix_web::{get, web, App, HttpServer, Result};

/// extract path info from "/users/{user_id}/{friend}" url
/// {user_id} - deserializes to a u32
/// {friend} - deserializes to a String
#[get("/users/{user_id}/{friend}")] // <- define path parameters
async fn index(path: web::Path<(u32, String)>) -> Result<String> {
    let (user_id, friend) = path.into_inner();
    Ok(format!("Welcome {}, user_id {}!", friend, user_id))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

It is also possible to extract path information to a type that implements the `Deserialize` trait from `serde` by matching dynamic segment names with field names. Here is an equivalent example that uses a deserialization struct using `serde` (make sure to enable its `derive` feature) instead of a tuple type.

```rust
use actix_web::{get, web, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    user_id: u32,
    friend: String,
}

/// extract path info using serde
#[get("/users/{user_id}/{friend}")] // <- define path parameters
async fn index(info: web::Path<Info>) -> Result<String> {
    Ok(format!(
        "Welcome {}, user_id {}!",
        info.friend, info.user_id
    ))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

As a non-type-safe alternative, it's also possible to query (see [`match_info` docs][docsrs_match_info]) the request for path parameters by name within a handler:

```rust
#[get("/users/{user_id}/{friend}")] // <- define path parameters
async fn index(req: HttpRequest) -> Result<String> {
    let name: String = req.match_info().get("friend").unwrap().parse().unwrap();
    let userid: i32 = req.match_info().query("user_id").parse().unwrap();

    Ok(format!("Welcome {}, user_id {}!", name, userid))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

## Query

The [`Query<T>`][querystruct] type provides extraction functionality for the request's query parameters. Underneath it uses `serde_urlencoded` crate.

```rust
use actix_web::{get, web, App, HttpServer};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

// this handler gets called if the query deserializes into `Info` successfully
// otherwise a 400 Bad Request error response is returned
#[get("/")]
async fn index(info: web::Query<Info>) -> String {
    format!("Welcome {}!", info.username)
}
```

## JSON

[`Json<T>`][jsonstruct] allows deserialization of a request body into a struct. To extract typed information from a request's body, the type `T` must implement `serde::Deserialize`.

```rust
use actix_web::{post, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body
#[post("/submit")]
async fn submit(info: web::Json<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}
```

Some extractors provide a way to configure the extraction process. To configure an extractor, pass its configuration object to the resource's `.app_data()` method. In the case of _Json_ extractor it returns a [_JsonConfig_][jsonconfig]. You can configure the maximum size of the JSON payload as well as a custom error handler function.

The following example limits the size of the payload to 4kb and uses a custom error handler.

```rust
use actix_web::{error, web, App, HttpResponse, HttpServer, Responder};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body, max payload size is 4kb
async fn index(info: web::Json<Info>) -> impl Responder {
    format!("Welcome {}!", info.username)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let json_config = web::JsonConfig::default()
            .limit(4096)
            .error_handler(|err, _req| {
                // create custom error response
                error::InternalError::from_response(err, HttpResponse::Conflict().finish())
                    .into()
            });

        App::new().service(
            web::resource("/")
                // change json extractor configuration
                .app_data(json_config)
                .route(web::post().to(index)),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## URL-Encoded Forms

A URL-encoded form body can be extracted to a struct, much like `Json<T>`. This type must implement `serde::Deserialize`.

[_FormConfig_][formconfig] allows configuring the extraction process.

```rust
use actix_web::{post, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct FormData {
    username: String,
}

/// extract form data using serde
/// this handler gets called only if the content type is *x-www-form-urlencoded*
/// and the content of the request could be deserialized to a `FormData` struct
#[post("/")]
async fn index(form: web::Form<FormData>) -> Result<String> {
    Ok(format!("Welcome {}!", form.username))
}
```

## Other

Actix Web also provides many other extractors, here's a few important ones:

- [`Data`][datastruct] - For accessing pieces of application state.
- [`HttpRequest`][httprequest] - `HttpRequest` is itself an extractor, in case you need access to other parts of the request.
- `String` - You can convert a request's payload to a `String`. [_An example_][stringexample] is available in the rustdoc.
- [`Bytes`][bytes] - You can convert a request's payload into _Bytes_. [_An example_][bytesexample] is available in the rustdoc.
- [`Payload`][payload] - Low-level payload extractor primarily for building other extractors. [_An example_][payloadexample] is available in the rustdoc.

## Application State Extractor

Application state is accessible from the handler with the `web::Data` extractor; however, state is accessible as a read-only reference. If you need mutable access to state, it must be implemented.

Here is an example of a handler that stores the number of processed requests:

```rust
use actix_web::{web, App, HttpServer, Responder};
use std::cell::Cell;

#[derive(Clone)]
struct AppState {
    count: Cell<usize>,
}

async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!("count: {}", data.count.get())
}

async fn add_one(data: web::Data<AppState>) -> impl Responder {
    let count = data.count.get();
    data.count.set(count + 1);

    format!("count: {}", data.count.get())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = AppState {
        count: Cell::new(0),
    };

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(data.clone()))
            .route("/", web::to(show_count))
            .route("/add", web::to(add_one))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Although this handler will work, `data.count` will only count the number of requests handled _by each worker thread_. To count the number of total requests across all threads, one should use shared `Arc` and [atomics][atomics].

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use std::{
    cell::Cell,
    sync::atomic::{AtomicUsize, Ordering},
    sync::Arc,
};

#[derive(Clone)]
struct AppState {
    local_count: Cell<usize>,
    global_count: Arc<AtomicUsize>,
}

#[get("/")]
async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[get("/add")]
async fn add_one(data: web::Data<AppState>) -> impl Responder {
    data.global_count.fetch_add(1, Ordering::Relaxed);

    let local_count = data.local_count.get();
    data.local_count.set(local_count + 1);

    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = AppState {
        local_count: Cell::new(0),
        global_count: Arc::new(AtomicUsize::new(0)),
    };

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(data.clone()))
            .service(show_count)
            .service(add_one)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

**Note**: If you want the _entire_ state to be shared across all threads, use `web::Data` and `app_data` as described in [Shared Mutable State][shared_mutable_state].

Be careful when using blocking synchronization primitives like `Mutex` or `RwLock` within your app state. Actix Web handles requests asynchronously. It is a problem if the [_critical section_][critical_section] in your handler is too big or contains an `.await` point. If this is a concern, we would advise you to also read [Tokio's advice on using blocking `Mutex` in async code][tokio_std_mutex].

[pathstruct]: https://docs.rs/actix-web/4/actix_web/dev/struct.Path.html
[querystruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Query.html
[jsonstruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Json.html
[jsonconfig]: https://docs.rs/actix-web/4/actix_web/web/struct.JsonConfig.html
[formconfig]: https://docs.rs/actix-web/4/actix_web/web/struct.FormConfig.html
[datastruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Data.html
[httprequest]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html
[stringexample]: https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html#impl-FromRequest-for-String
[bytes]: https://docs.rs/actix-web/4/actix_web/web/struct.Bytes.html
[bytesexample]: https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html#impl-FromRequest-5
[payload]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[payloadexample]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[docsrs_match_info]: https://docs.rs/actix-web/latest/actix_web/struct.HttpRequest.html#method.match_info
[actix]: /actix/docs/
[atomics]: https://doc.rust-lang.org/std/sync/atomic/
[shared_mutable_state]: application.md#shared-mutable-state
[critical_section]: https://en.wikipedia.org/wiki/Critical_section
[tokio_std_mutex]: https://tokio.rs/tokio/tutorial/shared-state#on-using-stdsyncmutex
