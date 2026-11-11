# Actix Web Integration

authbox works with any web framework. A complete, working Actix Web project lives in the `examples` directory of the repository. Check it for the full code. This page explains the main parts.

---

## Project layout

```text
src/
  main.rs             app state and server setup
  auth_middleware.rs  protects routes with is_token_valid
  common.rs           test fixtures: user store, templates, build_test_auth
  email_providers.rs  Resend, SendGrid and SMTP helpers
  handlers/           one file per endpoint
  models.rs           request bodies
  routes.rs           route configuration
```

## App state

Build the service once and share it through `web::Data`.

```rust
use actix_web::{App, HttpServer, dev::Server, web};
use authbox::prelude::AuthService;

pub struct AppState {
    pub auth: AuthService,
}

pub async fn server() -> Result<Server, std::io::Error> {
    // the redis stores are created asynchronously, so build the service first
    let auth = build_test_auth().await.map_err(std::io::Error::other)?;

    let state = web::Data::new(AppState { auth });

    let server = HttpServer::new(move || App::new().app_data(state.clone()).configure(config))
        .bind(("127.0.0.1", 8080))?
        .run();

    Ok(server)
}
```

## Authentication middleware

The middleware reads the bearer token and calls `is_token_valid`. Valid claims are stored in the request, so handlers can read them.

```rust
pub async fn auth_middleware<B>(
    req: ServiceRequest,
    next: Next<B>,
) -> Result<ServiceResponse<BoxBody>, Error>
where
    B: MessageBody + 'static,
{
    let state = req
        .app_data::<Data<AppState>>()
        .expect("AppState must be registered")
        .clone();

    let token = match req
        .headers()
        .get("Authorization")
        .and_then(|h| h.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "))
    {
        Some(token) => token.to_owned(),
        None => {
            return Ok(req.into_response(HttpResponse::Unauthorized().json(json!({
                "success": false,
                "message": "Missing bearer token"
            }))));
        }
    };

    let claims = match state.auth.is_token_valid(&token).await {
        Ok(claims) => claims,
        Err(_) => {
            return Ok(req.into_response(HttpResponse::Unauthorized().json(json!({
                "success": false,
                "message": "Invalid token"
            }))));
        }
    };

    // handlers can read these with web::ReqData<Claims>
    req.extensions_mut().insert(claims);

    let res = next.call(req).await?;
    Ok(res.map_into_boxed_body())
}
```

`is_token_valid` checks the signature, expiry, token type, blacklist and session cutoff. Refresh tokens are rejected here.

## Routes

Public routes sit at the top. Everything that needs a logged in user goes in a scope wrapped with the middleware.

```rust
pub fn config(cfg: &mut web::ServiceConfig) {
    cfg.service(index)
        .service(register)
        .service(login)
        .service(refresh)
        .service(verify_email)
        .service(forgot_password)
        .service(logout)
        .service(reset_password)
        .service(
            web::scope("/api")
                .wrap(from_fn(auth_middleware))
                .service(logout_all)
                .service(me),
        );
}
```

`logout` takes a refresh token in the body, so it does not need the middleware. `logout_all` and `me` need a logged in user.

## Handlers

A handler that needs the user reads the claims from the request and passes `claims.sub` to authbox:

```rust
#[post("/logout/all")]
pub async fn logout_all(
    state: web::Data<AppState>,
    claims: web::ReqData<Claims>,
) -> impl Responder {
    match state.auth.logout_all(&claims.sub).await {
        Ok(_) => HttpResponse::Ok().json(json!({
            "success": true,
            "message": "Logged out all sessions successfully"
        })),
        Err(e) => HttpResponse::Unauthorized().json(json!({
            "success": false,
            "error": format!("{:?}", e)
        })),
    }
}
```

A handler for a public endpoint calls the service directly:

```rust
#[post("/logout")]
pub async fn logout(state: web::Data<AppState>, body: web::Json<LogOutRequest>) -> impl Responder {
    match state.auth.logout(&body.refresh_token).await {
        Ok(_) => HttpResponse::Ok().json(json!({
            "success": true,
            "message": "Logged out successfully"
        })),
        Err(e) => HttpResponse::Unauthorized().json(json!({
            "success": false,
            "error": format!("{:?}", e)
        })),
    }
}
```

> The example returns `format!("{:?}", e)` to make development easy. In production, return a short message and map errors to status codes. See [Error Handling](errors.md).

## Email providers

`email_providers.rs` contains helpers that read credentials from environment variables.

```env
RESEND_API_KEY=...
RESEND_FROM_EMAIL=...
```

SendGrid and SMTP helpers work the same way. See [Configuring Email Delivery](email-provider.md).

## Running it

Start Redis, set your email environment variables, and run the example from its folder:

```bash
cargo run
```

The server listens on `127.0.0.1:8080`. The test email templates link to this address.

## Other frameworks

For Axum, Rocket or Warp the pattern is the same: keep one `AuthService` in shared state, call `is_token_valid` in your middleware or extractor, and pass `claims.sub` to the methods that take a `user_id`.
