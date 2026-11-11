# authbox

[![Crates.io](https://img.shields.io/crates/v/authbox.svg)](https://crates.io/crates/authbox)
[![Documentation](https://docs.rs/authbox/badge.svg)](https://docs.rs/authbox)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-support-orange?logo=kofi)](https://ko-fi.com/nyando)

A lightweight, modular, async-first authentication framework for Rust.

authbox is built around traits and pluggable components, so you keep your own database, cache and web framework. It handles the authentication logic for you.

Full guides and examples: [AuthBox Docs](https://authbox-docs.vercel.app/)

## Installation

```bash
cargo add authbox
```

## Features
 
* Argon2 password hashing, run on blocking threads so it never stalls the async runtime
* JWT access and refresh tokens
* Refresh token rotation (each refresh token can be used once)
* Refresh token revocation and blacklisting
* Session revocation: invalidate every token a user has issued so far
* Email verification flow
* Password reset flow with 15 minute reset tokens
* Change password for logged in users
* Account deletion with password confirmation
* Logout from a single device or from all devices
* One-time token (OTT) support with atomic claim, so a token cannot be used twice
* Optional user cache for the `me` lookup, with a default in-memory implementation (moka) and a `UserCache` trait for your own, plus hit and miss metrics
* Custom registration DTO support for extra profile fields
* Pluggable storage backends
* Framework agnostic
* Fully async (`tokio`)
* Builder API for ergonomic setup
* Test friendly design, with in-memory fixtures


## Quick start

```rust
use authbox::prelude::*;

let client = redis::Client::open("redis://127.0.0.1/")?;

let auth = AuthService::builder()
    .store(MyUserStore::new())                 // your database, implements UserStore
    .hasher(DefaultHasher)
    .tokens(DefaultJwtManager::new("a-secret-at-least-32-bytes-long!!", None, None)?)
    .blacklist(RedisBlacklistStore::new(client.clone()).await?)
    .email_sender(resend_email_provider())
    .email_templates(MyTemplates)              // implements EmailTemplateConfig
    .ott_store(RedisOttStore::new(client))
    .user_cache(DefaultMemoryUserCache::default()) // optional
    .build()?;

auth.register(&dto).await?;                            // sends a verification email
auth.verify_email(&token).await?;
let pair = auth.login("user@example.com", "a-long-password").await?;
let me = auth.me(&pair.access_token).await?;
let pair = auth.refresh_token(&pair.refresh_token).await?;
auth.logout(&pair.refresh_token).await?;
```

Other methods: `is_token_valid`, `request_password_reset`, `reset_password`, `change_password`, `logout_all`, `delete_account` and `invalidate_user_cache`.

In your auth middleware, call `is_token_valid()` to protect routes. For a complete Actix Web setup, check the `examples` directory.

## Things to know

* Passwords must be 12 to 128 characters.
* Password reset, password change, account deletion and `logout_all` revoke every existing session.
* Your database needs a UNIQUE index on the email column.
* Always take `user_id` from validated claims, never from request input.

## Support the project

If authbox saves you time, you can support its development on Ko-fi. It helps fund new features, documentation and maintenance.

[https://ko-fi.com/nyando](https://ko-fi.com/nyando)

## License

Apache-2.0