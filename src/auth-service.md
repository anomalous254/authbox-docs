# Building the Auth Service

Once your components are ready, assemble them into an `AuthService`. It is the main entry point of authbox.

---

## Components

| Builder method | Component | Required |
|---|---|---|
| `.store(..)` | `UserStore` | Yes |
| `.hasher(..)` | `PasswordHasher` | Yes |
| `.tokens(..)` | `TokenManager` | Yes |
| `.blacklist(..)` | `TokenBlacklistStore` | Yes |
| `.email_sender(..)` | `EmailProvider` | Yes |
| `.email_templates(..)` | `EmailTemplateConfig` | Yes |
| `.ott_store(..)` | `OneTimeTokenStore` | Yes |
| `.user_cache(..)` | `UserCache` | No |

## Building

```rust
use authbox::prelude::*;

let client = redis::Client::open("redis://127.0.0.1/")?;

let auth = AuthService::builder()
    .store(MyUserStore::new())
    .hasher(DefaultHasher)
    .tokens(DefaultJwtManager::new("a-secret-at-least-32-bytes-long!!", None, None)?)
    .blacklist(RedisBlacklistStore::new(client.clone()).await?)
    .email_sender(smtp_email_provider())
    .email_templates(MyTemplates)
    .ott_store(RedisOttStore::new(client))
    .user_cache(DefaultMemoryUserCache::default())
    .build()?;
```

`build()` returns a `Result`. It fails if a required component is missing.

`AuthService` is not generic, so you do not need a long type alias. Store it directly in your app state. It is cheap to clone.

## A reusable builder function

Wrap the setup in one function so your app and your tests share it:

```rust
pub async fn build_auth() -> Result<AuthService, Box<dyn std::error::Error + Send + Sync>> {
    let client = redis::Client::open("redis://127.0.0.1/")?;

    let auth = AuthService::builder()
        .store(TestStore::new())
        .hasher(DefaultHasher)
        .tokens(DefaultJwtManager::new(
            "test-only-secret-at-least-32-bytes-long!!",
            None,
            None,
        )?)
        .blacklist(RedisBlacklistStore::new(client.clone()).await?)
        .email_sender(resend_email_provider())
        .email_templates(MockTemplates)
        .ott_store(RedisOttStore::new(client))
        .user_cache(DefaultMemoryUserCache::default())
        .build()?;

    Ok(auth)
}
```

Here `resend_email_provider()` is a small helper that reads `RESEND_API_KEY` and `RESEND_FROM_EMAIL` and returns a `ResendEmailProvider`. Do the same for SendGrid or SMTP, as shown in [Configuring Email Delivery](email-provider.md).

## Available operations

```rust
auth.register(&dto).await?;
auth.verify_email(&token).await?;
auth.login(email, password).await?;
auth.me(&access_token).await?;
auth.refresh_token(&refresh_token).await?;
auth.logout(&refresh_token).await?;
auth.logout_all(&user_id).await?;
auth.request_password_reset(email).await?;
auth.reset_password(&token, new_password).await?;
auth.change_password(&user_id, old_password, new_password).await?;
auth.delete_account(&user_id, password).await?;
auth.is_token_valid(&access_token).await?;
auth.invalidate_user_cache(&user_id).await?;
```

See the [overview](overview.md) for what each one does.

## Summary

* Build the service once at startup and share it.
* Required components are checked when you call `build()`.
* The user cache is the only optional component.
