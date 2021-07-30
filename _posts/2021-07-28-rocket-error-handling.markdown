---
layout: post
title:  "Rocket v0.5 Error Handling Best Practices"
excerpt: ""
date:   2020-07-28 12:36:00
categories: 
tags: rust rocket
---

A common question in [#rocket](https://kiwiirc.com/client/irc.libera.chat/#rocket) is how to go beyond the basics in error handling. Specifically, there's a bit of a jump that's needed to connect the dots between defining a custom error type (e.g. with [`thiserror`](https://github.com/dtolnay/thiserror)) and the [Responder trait](https://api.rocket.rs/v0.5-rc/rocket/response/trait.Responder.html).

We can put these together and write handlers with idiomatic early returns, e.g.

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("HTTP Error {source:?}")]
    Reqwest {
        #[from] source: reqwest::Error,
    },
    #[error("SerdeJson Error {source:?}")]
    SerdeJson {
        #[from] source: serde_json::Error,
    },
}


#[get("/hello")]
async fn hello() -> Result<Json<Todo>, Error> {
    // a  https://docs.rs/reqwest/0.11.4/reqwest/struct.Error.html
    let response = reqwest::get("https://jsonplaceholder.typicode.com/todos/1")
        .await?
        .text()
        .await?;

    // https://docs.serde.rs/serde_json/struct.Error.html
    let todo: Todo = serde_json::from_str(&response)?;
    Ok(Json(todo))
}
```

Setting aside the contrived logic, this is error handling as we'd expect in Rust -- we're leaning on `?` and its implicit `into` to map errors into our `Error` type. 

But, in a real-world application, we're not typically happy to simply render a 500 back to the user. How do we respond differently to different error conditions? How do we log an error into a monitoring service like Sentry?

Here (and as of 0.5), the best practice I've found is to implement `Responder` for your `Error` type

```rust
impl<'r, 'o: 'r> Responder<'r, 'o> for Error {
    fn respond_to(self, req: &'r Request<'_>) -> response::Result<'o> {
        // log `self` to your favored error tracker, e.g.
        // sentry::capture_error(&self);

        match self {
            // in our simplistic example, we're happy to respond with the default 500 responder in all cases 
            _ => Status::InternalServerError.respond_to(req)
        }
    }
}
```

With this approach, we have a natural place to centralize error tracking as well as flexibility in the responses we send from the server. Not bad!