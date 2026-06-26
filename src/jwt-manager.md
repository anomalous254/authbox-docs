# Configuring JWT Tokens

AuthBox handles token generation, verification, and refresh through the `TokenManager` trait.

You can use the built-in JWT implementation `DefaultJwtManager` or provide your own custom token strategy.

---

## TokenManager Trait

```rust
use async_trait::async_trait;

#[async_trait]
pub trait TokenManager {
    type Token;
    type Claims;
    type Error;

    async fn generate(&self, user_id: &str) -> Result<Self::Token, Self::Error>;
    async fn verify(&self, token: &str) -> Result<Self::Claims, Self::Error>;
    async fn refresh(&self, refresh_token: &str) -> Result<Self::Token, Self::Error>;
}
```

**Responsibilities of a token manager:**

- Generate tokens
- Verify tokens
- Refresh tokens


## Built-in JWT Manager

AuthBox provides a default JWT manager:

```rust
let tokens = DefaultJwtManager::new("super-secret-key");
```

- Generates **access** and **refresh** tokens
- Access tokens expire after 15 minutes
- Refresh tokens expire after 1 day
- Tokens use the standard `Bearer` type
- Supports token blacklisting


## Login Flow

After successful authentication:

```rust
let auth_tokens = token_manager.generate(&user.id()).await?;
```

Returned tokens:

```rust
pub struct AuthTokens {
    pub access_token: String,
    pub refresh_token: String,
    pub access_expires_at: i64,
    pub token_type: String,
}
```


## Refresh Flow

Refreshing tokens:

```rust
token_manager.verify(refresh_token).await?;
token_manager.refresh(refresh_token).await?;
```

- Verifies the refresh token
- Returns a new set of access and refresh tokens


## Claims Type

Each `TokenManager` defines a claims type:

```rust
type Claims;
```

The built-in JWT manager uses:

```rust
pub struct JwtClaims {
    pub sub: String,
    pub exp: i64,
    pub jti: String,
    pub token_type: TokenType,
}
```

## Blacklist Support

To use AuthBox’s token blacklist, your claims type **must implement** `BlacklistableClaims`:

```rust
pub trait BlacklistableClaims {
    fn jti(&self) -> &str;
    fn exp(&self) -> i64;
}
```

The built-in claims already implement it:

```rust
impl BlacklistableClaims for JwtClaims {
    fn jti(&self) -> &str {
        &self.jti
    }

    fn exp(&self) -> i64 {
        self.exp
    }
}
```

- `jti()` identifies the token uniquely
- `exp()` determines when it can be removed from the blacklist

---



## Using a Custom Token Manager

You can implement your own:

```rust
pub struct MyTokenManager;

#[async_trait]
impl TokenManager for MyTokenManager {
    type Token = AuthTokens;
    type Claims = MyClaims;
    type Error = MyError;

    async fn generate(&self, user_id: &str) -> Result<Self::Token, Self::Error> { todo!() }
    async fn verify(&self, token: &str) -> Result<Self::Claims, Self::Error> { todo!() }
    async fn refresh(&self, refresh_token: &str) -> Result<Self::Token, Self::Error> { todo!() }
}
```

If you want blacklist support, implement `BlacklistableClaims` on your claims type:

```rust
impl BlacklistableClaims for MyClaims {
    fn jti(&self) -> &str { &self.jti }
    fn exp(&self) -> i64 { self.exp }
}
```

---

## Summary

- **Default option:** `DefaultJwtManager` for standard JWT tokens.
- **Custom option:** Implement `TokenManager` for any token strategy.
- **Blacklist support:** Claims must implement `BlacklistableClaims`.
- Fully flexible and backend-agnostic.
