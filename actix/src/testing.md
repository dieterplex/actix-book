# Testing

Every application should be well tested. Actix Web provides tools to perform integration tests against your applications and unit test tools for custom extractors and middleware.

Actix Web provides a request builder type. [_TestRequest_][testrequest] implements a builder-like pattern. You can generate a `HttpRequest` instance with `to_http_request()` and call your handlers or extractors with it. Also see

## Integration Testing For Applications

There are a few methods for testing your application. Actix Web can be used to run the application with specific handlers in a real HTTP server.

`TestRequest::get()`, `TestRequest::post()` and other methods can be used to send requests to the test server.

To create a `Service` for testing, use the `test::init_service` method which accepts a regular `App` builder.

> Check the [API documentation][actixdocs] for more information.

```rust
#[cfg(test)]
mod tests {
    use actix_web::{http::header::ContentType, test, App};

    use super::*;

    #[actix_web::test]
    async fn test_index_get() {
        let app = test::init_service(App::new().service(index)).await;
        let req = test::TestRequest::default()
            .insert_header(ContentType::plaintext())
            .to_request();
        let resp = test::call_service(&app, req).await;
        assert!(resp.status().is_success());
    }

    #[actix_web::test]
    async fn test_index_post() {
        let app = test::init_service(App::new().service(index)).await;
        let req = test::TestRequest::post().uri("/").to_request();
        let resp = test::call_service(&app, req).await;
        assert!(resp.status().is_client_error());
    }
}
```

If you need more complex application configuration, testing should be very similar to creating the normal application. For example, you may need to initialize application state. Create an `App` with a `data` method and attach state just like you would from a normal application.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::{test, web, App};

    #[actix_web::test]
    async fn test_index_get() {
        let app = test::init_service(
            App::new()
                .app_data(web::Data::new(AppState { count: 4 }))
                .service(index),
        )
        .await;
        let req = test::TestRequest::get().uri("/").to_request();
        let resp: AppState = test::call_and_read_body_json(&app, req).await;

        assert_eq!(resp.count, 4);
    }
}
```

## Stream Response Testing

If you need to test stream generation, it would be enough to call [`into_parts()`][resintoparts] and convert the resulting body into a future and execute it, for example when testing [_Server Sent Events_][serversentevents].

```rust
use std::task::Poll;

use actix_web::{
    http::{self, header::ContentEncoding, StatusCode},
    web, App, Error, HttpRequest, HttpResponse,
};
use futures::stream;

async fn sse(_req: HttpRequest) -> HttpResponse {
    let mut counter: usize = 5;

    // yields `data: N` where N in [5; 1]
    let server_events =
        stream::poll_fn(move |_cx| -> Poll<Option<Result<web::Bytes, Error>>> {
            if counter == 0 {
                return Poll::Ready(None);
            }
            let payload = format!("data: {}\n\n", counter);
            counter -= 1;
            Poll::Ready(Some(Ok(web::Bytes::from(payload))))
        });

    HttpResponse::build(StatusCode::OK)
        .insert_header((http::header::CONTENT_TYPE, "text/event-stream"))
        .insert_header(ContentEncoding::Identity)
        .streaming(server_events)
}

pub fn main() {
    App::new().route("/", web::get().to(sse));
}

#[cfg(test)]
mod tests {
    use super::*;

    use actix_web::{body, body::MessageBody as _, rt::pin, test, web, App};
    use futures::future;

    #[actix_web::test]
    async fn test_stream_chunk() {
        let app = test::init_service(App::new().route("/", web::get().to(sse))).await;
        let req = test::TestRequest::get().to_request();

        let resp = test::call_service(&app, req).await;
        assert!(resp.status().is_success());

        let body = resp.into_body();
        pin!(body);

        // first chunk
        let bytes = future::poll_fn(|cx| body.as_mut().poll_next(cx)).await;
        assert_eq!(
            bytes.unwrap().unwrap(),
            web::Bytes::from_static(b"data: 5\n\n")
        );

        // second chunk
        let bytes = future::poll_fn(|cx| body.as_mut().poll_next(cx)).await;
        assert_eq!(
            bytes.unwrap().unwrap(),
            web::Bytes::from_static(b"data: 4\n\n")
        );

        // remaining part
        for i in 0..3 {
            let expected_data = format!("data: {}\n\n", 3 - i);
            let bytes = future::poll_fn(|cx| body.as_mut().poll_next(cx)).await;
            assert_eq!(bytes.unwrap().unwrap(), web::Bytes::from(expected_data));
        }
    }

    #[actix_web::test]
    async fn test_stream_full_payload() {
        let app = test::init_service(App::new().route("/", web::get().to(sse))).await;
        let req = test::TestRequest::get().to_request();

        let resp = test::call_service(&app, req).await;
        assert!(resp.status().is_success());

        let body = resp.into_body();
        let bytes = body::to_bytes(body).await;
        assert_eq!(
            bytes.unwrap(),
            web::Bytes::from_static(b"data: 5\n\ndata: 4\n\ndata: 3\n\ndata: 2\n\ndata: 1\n\n")
        );
    }
}
```

## Unit Testing Extractors

Unit testing has pretty limited value for applications, but can be useful when developing extractors, middleware, and responders. Given that, calling directly into handler functions **which are defined stand-alone, without using routing macros** (like `#[get("/")]`) is possible if you want to make assertions on custom `Responder`s.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::{
        http::{self, header::ContentType},
        test,
    };

    #[actix_web::test]
    async fn test_index_ok() {
        let req = test::TestRequest::default()
            .insert_header(ContentType::plaintext())
            .to_http_request();
        let resp = index(req).await;
        assert_eq!(resp.status(), http::StatusCode::OK);
    }

    #[actix_web::test]
    async fn test_index_not_ok() {
        let req = test::TestRequest::default().to_http_request();
        let resp = index(req).await;
        assert_eq!(resp.status(), http::StatusCode::BAD_REQUEST);
    }
}
```

[serversentevents]: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
[resintoparts]: https://docs.rs/actix-web/4/actix_web/struct.HttpResponse.html#method.into_parts
[actixdocs]: https://docs.rs/actix-web/4/actix_web/test/index.html
[testrequest]: https://docs.rs/actix-web/4/actix_web/test/struct.TestRequest.html
