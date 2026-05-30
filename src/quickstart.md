# Quick Start

> **Note**
>
> This quick start is intended to demonstrate the overall authentication workflow.
> Some code has been omitted for brevity. Refer to the next chapters for complete implementations and production-ready examples.

This example shows:

1. Create a user type
2. Create a store
3. Configure authentication
4. Register a user
5. Login
6. Refresh tokens

## User

```rust
#[derive(Clone, Debug)]
pub struct User {
    // Required fileds
    pub id: String,
    pub email: String,
    pub password_hash: String,
    pub is_email_verified: bool,

    // Your custom fileds
    pub username : String,
    pub location: String,
    ...
    
}
```

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

Build the service:

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

Register:

```rust
let user = auth.register(dto).await?;
```

Login:

```rust
let tokens = auth
    .login(email, password)
    .await?;
```

Refresh:

```rust
let refreshed = auth
    .refresh_token(
        &tokens.refresh_token
    )
    .await?;
```
