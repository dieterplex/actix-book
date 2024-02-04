# Hosting on Shuttle

<img width="300" src="https://raw.githubusercontent.com/shuttle-hq/shuttle/master/assets/logo-rectangle-transparent.png"/>

> [**Shuttle**](https://www.shuttle.rs) is a Rust-native cloud development platform that lets you deploy your Rust apps for free.

Shuttle has out-of-the-box support for Actix Web. Follow these steps to host your web service on Shuttle:

1. Add Shuttle dependencies to `Cargo.toml`:

```toml
[dependencies]
actix-web = "4"
shuttle-actix-web = "0.36"
shuttle-runtime = "0.36"
tokio = "1"
```

2. Add the `#[shuttle_runtime::main]` annotation and update the `main` function as follows:

```rust
use actix_web::{get, web::ServiceConfig};
use shuttle_actix_web::ShuttleActixWeb;

#[shuttle_runtime::main]
async fn main() -> ShuttleActixWeb<impl FnOnce(&mut ServiceConfig) + Send + Clone + 'static> {
    let config = move |cfg: &mut ServiceConfig| {
        // set up your service here, e.g.:
        cfg.service(hello_world);
    };

    Ok(config.into())
}
```

3. Install `cargo-shuttle`:

```sh
cargo install cargo-shuttle
```

4. Create your project on the Shuttle platform:

```sh
cargo shuttle project start
```

5. Deploy! 🚀

```sh
cargo shuttle deploy
```

You can run `cargo shuttle run` to test your application locally.

Check out some complete Actix Web examples [here](https://github.com/shuttle-hq/shuttle-examples/tree/main/actix-web).
