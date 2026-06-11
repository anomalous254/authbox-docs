# authbox

[![Crates.io](https://img.shields.io/crates/v/authbox.svg)](https://crates.io/crates/authbox)
[![Documentation](https://docs.rs/authbox/badge.svg)](https://docs.rs/authbox)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-support-orange?logo=kofi)](https://ko-fi.com/nyando)

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
>    - Database
>    - ORM
>    - Web framework
>
> 2. It works with:
>    - Axum
>    - Actix
>    - Warp
>    - Rocket
>    - Custom applications


# Support the Project

If AuthBox helps you build applications faster, consider supporting its development.

Your support helps fund:

- New features
- Documentation improvements
- Bug fixes and maintenance
- Long-term project sustainability

❤️ Ko-fi: [https://ko-fi.com/nyando](https://ko-fi.com/nyando)

Every contribution, no matter the size, is greatly appreciated.

Thank you for supporting open-source software!