# Individual file

It is possible to serve static files with a custom path pattern and `NamedFile`. To match a path tail, we can use a `[.*]` regex.

```rust
use actix_files::NamedFile;
use actix_web::{HttpRequest, Result};
use std::path::PathBuf;

async fn index(req: HttpRequest) -> Result<NamedFile> {
    let path: PathBuf = req.match_info().query("filename").parse().unwrap();
    Ok(NamedFile::open(path)?)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| App::new().route("/{filename:.*}", web::get().to(index)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

<div class="warning">
WARNING

Matching a path tail with the `[.*]` regex and using it to return a `NamedFile` has serious security implications. It offers the possibility for an attacker to insert `../` into the URL and access every file on the host that the user running the server has access to.
</div>

## Directory

To serve files from specific directories and sub-directories, [`Files`][files] can be used. `Files` must be registered with an `App::service()` method, otherwise it will be unable to serve sub-paths.

```rust
use actix_files as fs;
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(fs::Files::new("/static", ".").show_files_listing()))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

By default files listing for sub-directories is disabled. Attempt to load directory listing will return _404 Not Found_ response. To enable files listing, use [`Files::show_files_listing()`][showfileslisting] method.

Instead of showing files listing for a directory, it is possible to redirect to a specific index file. Use the [`Files::index_file()`][indexfile] method to configure this redirect.

## Configuration

`NamedFiles` can specify various options for serving files:

- `set_content_disposition` - function to be used for mapping file's mime to corresponding `Content-Disposition` type
- `use_etag` - specifies whether `ETag` shall be calculated and included in headers.
- `use_last_modified` - specifies whether file modified timestamp should be used and added to `Last-Modified` header.

All of the above methods are optional and provided with the best defaults, But it is possible to customize any of them.

```rust

use actix_files as fs;
use actix_web::http::header::{ContentDisposition, DispositionType};
use actix_web::{get, App, Error, HttpRequest, HttpServer};

#[get("/{filename:.*}")]
async fn index(req: HttpRequest) -> Result<fs::NamedFile, Error> {
    let path: std::path::PathBuf = req.match_info().query("filename").parse().unwrap();
    let file = fs::NamedFile::open(path)?;
    Ok(file
        .use_last_modified(true)
        .set_content_disposition(ContentDisposition {
            disposition: DispositionType::Attachment,
            parameters: vec![],
        }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

The Configuration can also be applied to directory service:

```rust
use actix_files as fs;
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            fs::Files::new("/static", ".")
                .show_files_listing()
                .use_last_modified(true),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

[files]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#
[showfileslisting]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#method.show_files_listing
[indexfile]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#method.index_file
