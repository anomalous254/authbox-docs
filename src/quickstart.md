# Quick Start

This page walks through the whole workflow in a few minutes. Later chapters cover each piece in detail.

You will:

1. Create a user type and a registration DTO
2. Implement a `UserStore`
3. Build an `AuthService`
4. Register, verify, log in, refresh and log out

> **Note**
>
> The user store and email templates are your own code, so they are shown only in outline here. The chapters on [user stores](user-store.md) and [email templates](email-templates.md) have full examples, and the `examples` directory has a working Actix Web project.

## 1. Build the service

```rust
use authbox::prelude::*;

let client = redis::Client::open("redis://127.0.0.1/")?;

let auth = AuthService::builder()
    .store(MyUserStore::new())                      // your database, implements UserStore
    .hasher(DefaultHasher)
    .tokens(DefaultJwtManager::new("a-secret-at-least-32-bytes-long!!", None, None)?)
    .blacklist(RedisBlacklistStore::new(client.clone()).await?)
    .email_sender(smtp_email_provider())            // or Resend, or SendGrid
    .email_templates(MyTemplates)                   // implements EmailTemplateConfig
    .ott_store(RedisOttStore::new(client))
    .user_cache(DefaultMemoryUserCache::default())  // optional
    .build()?;
```

`AuthService` is cheap to clone. Build it once at startup and share it with your handlers.

## 2. Register

```rust
let user = auth.register(&dto).await?;
```

This checks the password length, hashes the password, creates the user through your store, stores a verification token and sends the verification email. If the token or the email fails, the new user is rolled back.

## 3. Verify the email

```rust
auth.verify_email(&token).await?;
```

The token comes from the link in the email. It works once.

## 4. Log in

```rust
let pair = auth.login("user@example.com", "a-long-password").await?;
```

You get a `TokenPair` with an `access_token` and a `refresh_token`.

## 5. Read the current user

```rust
let me = auth.me(&pair.access_token).await?;
```

## 6. Refresh

```rust
let pair = auth.refresh_token(&pair.refresh_token).await?;
```

The old refresh token is revoked and a new pair is issued. Using the old token again fails.

## 7. Log out

```rust
// this device only
auth.logout(&pair.refresh_token).await?;
```

## What else is there

| Method | Chapter |
|---|---|
| `is_token_valid` | [Actix Web Integration](actix-web.md) |
| `logout_all` | [Sessions and Revocation](sessions.md) |
| `request_password_reset`, `reset_password` | [Password and Account Management](account-management.md) |
| `change_password`, `delete_account` | [Password and Account Management](account-management.md) |
| `invalidate_user_cache` | [Configuring the User Cache](user-cache.md) |

## Next steps

Continue with [Creating a Registration DTO](registration-dto.md) and work down the Building Authentication chapters.
