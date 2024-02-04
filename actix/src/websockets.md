# Websockets

Actix Web supports WebSockets with the `actix-web-actors` crate. It is possible to convert a request's `Payload` to a stream of [_ws::Message_][message] with a [_web::Payload_][payload] and then use stream combinators to handle actual messages, but it is simpler to handle websocket communications with an http actor.

The following is an example of a simple websocket echo server:

```rust
use actix::{Actor, StreamHandler};
use actix_web::{web, App, Error, HttpRequest, HttpResponse, HttpServer};
use actix_web_actors::ws;

/// Define HTTP actor
struct MyWs;

impl Actor for MyWs {
    type Context = ws::WebsocketContext<Self>;
}

/// Handler for ws::Message message
impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for MyWs {
    fn handle(&mut self, msg: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match msg {
            Ok(ws::Message::Ping(msg)) => ctx.pong(&msg),
            Ok(ws::Message::Text(text)) => ctx.text(text),
            Ok(ws::Message::Binary(bin)) => ctx.binary(bin),
            _ => (),
        }
    }
}

async fn index(req: HttpRequest, stream: web::Payload) -> Result<HttpResponse, Error> {
    let resp = ws::start(MyWs {}, &req, stream);
    println!("{:?}", resp);
    resp
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/ws/", web::get().to(index)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

> A simple websocket echo server example is available in the [examples directory][examples].

> An example chat server with the ability to chat over a websocket or TCP connection is available in [websocket-chat directory][chat]

[message]: https://docs.rs/actix-web-actors/2/actix_web_actors/ws/enum.Message.html
[payload]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[examples]: https://github.com/actix/examples/tree/master/websockets
[chat]: https://github.com/actix/examples/tree/master/websockets/chat
