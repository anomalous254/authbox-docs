# Architecture

authbox is a trait driven authentication framework. Every core job sits behind a small interface, and `AuthService` ties them together.

## Crates

The repository is a workspace of small crates.

| Crate | Purpose |
|---|---|
| `authbox` | The main crate. Re-exports everything through `authbox::prelude` |
| `authbox-core` | Traits, types, errors and the `AuthService` engine |
| `authbox-jwt` | `DefaultJwtManager`, the JWT access and refresh token provider |
| `authbox-argon2` | `DefaultHasher`, Argon2 password hashing |
| `authbox-redis` | `RedisBlacklistStore` and `RedisOttStore` |
| `authbox-cache` | `DefaultMemoryUserCache`, an in-memory user cache built on moka |
| `authbox-email` | `ResendEmailProvider`, `SendGridEmailProvider` and `SmtpEmailProvider` |

## Design principles

* **Separation of concerns.** Authentication logic is kept apart from storage, email and token handling.
* **Dependency injection.** Everything is composed through `AuthService::builder()`.
* **Backend agnostic.** Use any database, cache, email service or web framework.
* **Testable.** Every component can be replaced with a mock or in-memory version.
* **Cheap to share.** `AuthService` wraps its parts in an `Arc`, so cloning it is inexpensive.

## Components

| Layer | Trait | Default |
|---|---|---|
| Users | `UserStore` | None, you implement it |
| Passwords | `PasswordHasher` | `DefaultHasher` |
| Sessions | `TokenManager` | `DefaultJwtManager` |
| Revocation | `TokenBlacklistStore` | `RedisBlacklistStore` |
| Temporary tokens | `OneTimeTokenStore` | `RedisOttStore` |
| Caching (optional) | `UserCache` | `DefaultMemoryUserCache` |
| Delivery | `EmailProvider` | Resend, SendGrid, SMTP |
| Content | `EmailTemplateConfig` | None, you implement it |

## Request flow

1. Your web framework receives a request and calls a method on `AuthService`.
2. `AuthService` uses `PasswordHasher` for hashing and checking passwords.
3. It uses `UserStore` to read and write users.
4. It uses `TokenManager` to create and verify JWTs.
5. It uses `TokenBlacklistStore` to revoke refresh tokens and whole sessions.
6. It uses `OneTimeTokenStore` for verification and reset tokens.
7. It uses `EmailProvider` and `EmailTemplateConfig` to send emails.
8. If configured, it uses `UserCache` to speed up `me()`.

## Key idea

authbox does not hard code how users are stored or how emails are sent. It runs the security rules and calls your components to do the actual work.
