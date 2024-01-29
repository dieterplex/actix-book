## Installing Rust

If you don't have Rust yet, we recommend you use `rustup` to manage your Rust installation. The [official rust guide][rustguide] has a wonderful section on getting started.

<p>
Actix Web currently has a minimum supported Rust version (MSRV) of 1.59. Running <code>rustup update</code> will ensure you have the latest and greatest Rust version available. As such, this guide assumes you are running Rust 1.59 or later.
</p>

## Hello, world!

Start by creating a new binary-based Cargo project and changing into the new directory:

```bash
cargo new hello-world
cd hello-world
```

Add `actix-web` as a dependency of your project by adding the following to your `Cargo.toml` file.

<!-- DEPENDENCY -->

```toml
[dependencies]
actix-web = "4"
```

Request handlers use async functions that accept zero or more parameters. These parameters can be extracted from a request (see `FromRequest` trait) and returns a type that can be converted into an `HttpResponse` (see `Responder` trait):

Replace the contents of `src/main.rs` with the following:

```rust
use actix_web::{get, post, web, App, HttpResponse, HttpServer, Responder};

#[get("/")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().body("Hello world!")
}

#[post("/echo")]
async fn echo(req_body: String) -> impl Responder {
    HttpResponse::Ok().body(req_body)
}

async fn manual_hello() -> impl Responder {
    HttpResponse::Ok().body("Hey there!")
}
```

Notice that some of these handlers have routing information attached directly using the built-in macros. These allow you to specify the method and path that the handler should respond to. You will see below how to register `manual_hello` (i.e. routes that do not use a routing macro).

Next, create an `App` instance and register the request handlers. Use `App::service` for the handlers using routing macros and `App::route` for manually routed handlers, declaring the path and method. Finally, the app is started inside an `HttpServer` which will serve incoming requests using your `App` as an "application factory".

Further append the following `main` function to `src/main.rs`:

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(hello)
            .service(echo)
            .route("/hey", web::get().to(manual_hello))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```
That's it! Compile and run the program with `cargo run`. The `#[actix_web::main]` macro executes the async main function within the actix runtime. Now you can go to `http://127.0.0.1:8080/` or any of the other routes you defined to see the results.

<!-- LINKS -->

[rustguide]: https://doc.rust-lang.org/book/ch01-01-installation.html
[actix-web-codegen]: https://docs.rs/actix-web-codegen/
