# authbox

A lightweight, modular, async-first authentication framework for Rust.

`authbox` provides a flexible authentication system built around traits, pluggable components, and Tokio-ready async APIs.

It is designed for applications that need customizable authentication logic without being locked into a specific database, framework, or storage backend.

---

# Features

- Secure password hashing with Argon2
- JWT access + refresh token authentication
- Refresh token rotation
- Refresh token revocation / blacklisting
- Email verification flow
- Password reset flow
- One-time token (OTT) support
- Fully async (`tokio`)
- Trait-driven architecture
- Pluggable storage backends
- Framework agnostic
- Test-friendly design
- Builder API for ergonomic setup
- Custom registration DTO support


> 1. Unlike full authentication platforms, `authbox` does not require a specific:
> - Database
> - ORM
> - Web framework
> 2. It works with:
> - Axum
> - Actix
> - Warp
> - Rocket
> - Custom applications
