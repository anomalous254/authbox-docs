# Building the Auth Service

After configuring all the components, they can be assembled into an `AuthService`.

The `AuthService` is the **main entry point** into AuthBox and provides all authentication functionality.

---

## Service Components

An `AuthService` requires the following pluggable components:

- `UserStore` – persistence for user accounts  
- `PasswordHasher` – for hashing and verifying passwords  
- `TokenManager` – for access and refresh tokens  
- `TokenBlacklistStore` – for managing blacklisted refresh tokens  
- `EmailProvider` – for sending emails  
- `EmailTemplateConfig` – for generating email subjects and bodies  
- `OneTimeTokenStore` – for temporary tokens (email verification, password reset, magic links)

---

## Building an AuthService

```rust
let auth = AuthService::builder()
    .store(UserStoreImpl::new())
    .hasher(DefaultHasher)
    .tokens(DefaultJwtManager::new("secret"))
    .blacklist(MemoryBlacklistStore::new())
    .email_sender(MockEmailSender)
    .email_templates(MockTemplates)
    .ott_store(MemoryOttStore::new())
    .build();
```

- Each component is pluggable and can be replaced with a custom implementation.  
- This ensures the AuthBox service is **flexible** and backend-agnostic.

---

## Available Operations

After constructing the service, it exposes methods for the complete authentication workflow:

```rust
auth.register(...);
auth.login(...);
auth.logout(...);
auth.refresh_token(...);
auth.verify_email(...);
auth.request_password_reset(...);
auth.reset_password(...);
auth.is_token_valid(...); // use in auth middleware to verify access tokens
```

- These methods integrate all components automatically (user store, tokens, email, templates, etc.).

---

## Example: Test AuthService

You can build a test instance using Redis, in-memory, and mock components:

```rust
pub type TestAuthService = AuthService<
    TestStore,
    DefaultHasher,
    DefaultJwtManager,
    RedisBlacklistStore,
    MockEmailSender,
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
        .email_sender(MockEmailSender)
        .email_templates(MockTemplates)
        .ott_store(RedisOttStore::new(client))
        .build()
}
```

- Ideal for **unit testing** or **local development**.  
- External dependencies are minimized; storage and email operations are handled by in-memory or mock components.

---

## Summary

- `AuthService` is the **central interface** for AuthBox.  
- All components are **fully pluggable**, allowing complete customization.  
- Supports registration, login, email verification, password resets, token rotation, and one-time authentication flows.  
- Testable with mock and in-memory implementations for fast local development.