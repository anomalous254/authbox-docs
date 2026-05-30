# Building the Auth Service

After configuring all the components, they can be assembled into an `AuthService`.

The `AuthService` is the **main entry point** into AuthBox and provides all authentication functionality.

---

## Service Components

An `AuthService` requires the following components:

- `UserStore` – persistence for user accounts
- `PasswordHasher` – for hashing and verifying passwords
- `TokenManager` – for access and refresh tokens
- `TokenBlacklistStore` – for managing blacklisted refresh tokens
- `EmailProvider` – for sending emails
- `EmailTemplateConfig` – for generating email subjects and bodies
- `OneTimeTokenStore` – for temporary tokens (email verification, password reset, magic links)


## Building an AuthService

```rust
let auth =
    AuthService::builder()
        .store(UserStoreImpl::new())
        .hasher(DefaultHasher)
        .tokens(
            DefaultJwtManager::new("secret")
        )
        .blacklist(
            MemoryBlacklistStore::new()
        )
        .email_sender(
            MockEmailSender
        )
        .email_templates(
            MockTemplates
        )
        .ott_store(
            MemoryOttStore::new()
        )
        .build();
```

- Each component is pluggable and can be swapped with a custom implementation.
- This ensures the AuthBox service is flexible and backend-agnostic.


## Available Operations

After constructing the service, it exposes methods for the complete authentication workflow:

```rust
auth.register(...)
auth.login(...)
auth.refresh_token(...)
auth.verify_email(...)
auth.request_password_reset(...)
auth.reset_password(...)
```

- These methods integrate all components automatically (user store, tokens, email, templates, etc.)


## Example: Test AuthService

You can easily build a test instance using in-memory and mock components:

```rust
type TestAuthService = AuthService<
    TestStore,
    DefaultHasher,
    DefaultJwtManager,
    MemoryBlacklistStore,
    MockEmailSender,
    MockTemplates,
    MemoryOttStore,
>;

pub fn build_test_auth() -> TestAuthService {
    AuthService::builder()
        .store(TestStore::new())
        .hasher(DefaultHasher)
        .tokens(DefaultJwtManager::new("secret"))
        .blacklist(MemoryBlacklistStore::new())
        .email_sender(MockEmailSender)
        .email_templates(MockTemplates)
        .ott_store(MemoryOttStore::new())
        .build()
}
```

- Ideal for **unit testing** or **local development**.
- No external dependencies are required; all storage and email operations are in-memory or mocked.


## Summary

- `AuthService` is the central interface for AuthBox.
- All configurable components are **pluggable**, allowing full customization.
- Supports registration, login, email verification, password resets, token rotation, and one-time authentication flows.
- Can be fully tested with mock and in-memory implementations.
