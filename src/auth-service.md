# Building the Auth Service

After configuring all required components, assemble them into an `AuthService`.

The `AuthService` is the **main entry point** into AuthBox and coordinates authentication, email delivery, token management, and user persistence.

---

## Required Components

An `AuthService` is built from the following pluggable components:

- `UserStore` — persistence layer for user accounts
- `PasswordHasher` — hashes and verifies passwords
- `TokenManager` — creates and validates access/refresh tokens
- `TokenBlacklistStore` — stores revoked refresh tokens
- `EmailProvider` — sends authentication emails
- `EmailTemplateConfig` — generates email subjects and bodies
- `OneTimeTokenStore` — stores temporary tokens for email verification, password resets, and magic links

Each component can be replaced with your own implementation.

---

## Building an AuthService

The example below uses:

- `TestStore`
- `DefaultHasher`
- `DefaultJwtManager`
- `RedisBlacklistStore`
- `SmtpEmailProvider`
- `MockTemplates`
- `RedisOttStore`

```rust
use authbox::prelude::*;

let client = redis::Client::open("redis://127.0.0.1/")
    .expect("failed to connect to redis");

let auth = AuthService::builder()
    .store(TestStore::new())
    .hasher(DefaultHasher)
    .tokens(DefaultJwtManager::new("secret"))
    .blacklist(RedisBlacklistStore::new(client.clone()))
    .email_sender(smtp_email_provider())
    .email_templates(MockTemplates)
    .ott_store(RedisOttStore::new(client))
    .build();
```

---

## SMTP Configuration

The SMTP provider can be created from environment variables.

```rust
use std::env;
use authbox::prelude::SmtpEmailProvider;

pub fn smtp_email_provider() -> SmtpEmailProvider {
    let host = env::var("SMTP_HOST")
        .expect("SMTP_HOST missing");

    let username = env::var("SMTP_USERNAME")
        .expect("SMTP_USERNAME missing");

    let password = env::var("SMTP_PASSWORD")
        .expect("SMTP_PASSWORD missing");

    let from_email = env::var("SMTP_FROM_EMAIL")
        .expect("SMTP_FROM_EMAIL missing");

    SmtpEmailProvider::new(
        &host,
        &username,
        &password,
        &from_email,
    )
}
```

Example environment variables:

```env
SMTP_HOST=smtp.gmail.com
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM_EMAIL=AuthBox <your-email@gmail.com>
```

> **Note:** For Gmail, use an App Password instead of your account password.

---

## Available Operations

Once constructed, `AuthService` provides the complete authentication workflow.

```rust
auth.register(...).await?;
auth.login(...).await?;
auth.logout(...).await?;
auth.refresh_token(...).await?;
auth.verify_email(...).await?;
auth.request_password_reset(...).await?;
auth.reset_password(...).await?;
auth.is_token_valid(...).await?;
```

These methods automatically coordinate the configured storage, password hashing, token management, one-time tokens, email templates, and email provider.

---

## Example Service Type

```rust
pub type TestAuthService = AuthService<
    TestStore,
    DefaultHasher,
    DefaultJwtManager,
    RedisBlacklistStore,
    SmtpEmailProvider,
    MockTemplates,
    RedisOttStore,
>;
```

---

## Complete Builder Example

```rust
pub fn build_test_auth() -> TestAuthService {
    let client = redis::Client::open("redis://127.0.0.1/")
        .expect("failed to connect to redis");

    AuthService::builder()
        .store(TestStore::new())
        .hasher(DefaultHasher)
        .tokens(DefaultJwtManager::new("secret"))
        .blacklist(RedisBlacklistStore::new(client.clone()))
        .email_sender(smtp_email_provider())
        .email_templates(MockTemplates)
        .ott_store(RedisOttStore::new(client))
        .build()
}
```

---

## Summary

- `AuthService` is the central interface for AuthBox.
- Every dependency is pluggable and can be replaced with your own implementation.
- SMTP, SendGrid, and Resend providers can be used interchangeably through the `EmailProvider` abstraction.
- Authentication workflows automatically integrate storage, hashing, JWTs, email delivery, templates, and one-time tokens.
- The builder pattern makes it easy to switch implementations for development, testing, and production.
