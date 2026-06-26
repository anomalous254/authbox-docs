# Quick Start

> **Note**
>
> This quick start demonstrates the overall authentication workflow using the built-in `SmtpEmailProvider`.
> Some implementation details have been omitted for brevity. The following chapters cover complete implementations for the user store, email templates, and production-ready configuration.

This example walks through:

1. Creating a user type
2. Implementing `AuthUser`
3. Building an `AuthService`
4. Registering a user
5. Logging in
6. Refreshing an access token

---

## User

```rust
#[derive(Clone, Debug)]
pub struct User {
    // Required fields
    pub id: String,
    pub email: String,
    pub password_hash: String,
    pub is_email_verified: bool,

    // Your application fields
    pub username: String,
    pub location: String,
}
```

Implement the `AuthUser` trait:

```rust
impl AuthUser for User {
    fn id(&self) -> String {
        self.id.clone()
    }

    fn email(&self) -> &str {
        &self.email
    }

    fn password_hash(&self) -> &str {
        &self.password_hash
    }

    fn is_email_verified(&self) -> bool {
        self.is_email_verified
    }

    fn set_email_verified(&mut self, verified: bool) {
        self.is_email_verified = verified;
    }

    fn set_password_hash(&mut self, hash: String) {
        self.password_hash = hash;
    }
}
```

---

## Build the AuthService

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

The `smtp_email_provider()` function creates a `SmtpEmailProvider` using the following environment variables:

```env
SMTP_HOST=smtp.gmail.com
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM_EMAIL=AuthBox <your-email@gmail.com>
```

---

## Register a User

```rust
let user = auth.register(dto).await?;
```

Registering a user will:

- Create the account
- Hash the password
- Generate an email verification token
- Send a verification email using the configured `SmtpEmailProvider`

---

## Login

```rust
let tokens = auth
    .login(email, password)
    .await?;
```

On success, AuthBox returns an access token and a refresh token.

---

## Refresh Tokens

```rust
let refreshed = auth
    .refresh_token(&tokens.refresh_token)
    .await?;
```

A new access token (and refresh token, if rotation is enabled) is generated while the previous refresh token is revoked.

---

## Next Steps

The following chapters explain how to customize:

- `UserStore`
- `EmailTemplateConfig`
- `SmtpEmailProvider`
- JWT configuration
- One-time token storage
- Redis integration
