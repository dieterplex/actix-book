`actix-web` automatically upgrades connections to _HTTP/2_ if possible.

# Negotiation

<!-- TODO: use rustls example -->

When either of the `rustls` or `openssl` features are enabled, `HttpServer` provides the [bind_rustls][bindrustls] method and [bind_openssl][bindopenssl] methods, respectively.

<!-- DEPENDENCY -->

```toml
[dependencies]
actix-web = { version = "4", features = ["openssl"] }
openssl = { version = "0.10", features = ["v110"] }
```

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

async fn index(_req: HttpRequest) -> impl Responder {
    "Hello."
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // load TLS keys
    // to create a self-signed temporary cert for testing:
    // `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'`
    let mut builder = SslAcceptor::mozilla_intermediate(SslMethod::tls()).unwrap();
    builder
        .set_private_key_file("key.pem", SslFiletype::PEM)
        .unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| App::new().route("/", web::get().to(index)))
        .bind_openssl("127.0.0.1:8080", builder)?
        .run()
        .await
}
```

Upgrades to HTTP/2 described in [RFC 7540 §3.2][rfcsection32] are not supported. Starting HTTP/2 with prior knowledge is supported for both cleartext and TLS connections ([RFC 7540 §3.4][rfcsection34]) (when using the lower level `actix-http` service builders).

> Check out [the TLS examples][examples] for concrete example.

[rfcsection32]: https://httpwg.org/specs/rfc7540.html#rfc.section.3.2
[rfcsection34]: https://httpwg.org/specs/rfc7540.html#rfc.section.3.4
[bindrustls]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_rustls
[bindopenssl]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_openssl
[tlsalpn]: https://tools.ietf.org/html/rfc7301
[examples]: https://github.com/actix/examples/tree/master/https-tls
