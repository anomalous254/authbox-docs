# Configuring JWT Tokens

authbox creates and verifies tokens through the `TokenManager` trait. Use the built-in `DefaultJwtManager` or write your own.

---

## The trait

```rust
#[async_trait]
pub trait TokenManager: Send + Sync {
    async fn generate(&self, user_id: &str) -> Result<TokenPair, BoxError>;
    async fn verify(&self, token: &str) -> Result<Claims, BoxError>;

    /// How long a refresh token lives, in seconds.
    fn refresh_ttl_secs(&self) -> u64;
}
```

Responsibilities:

* `generate` creates an access token and a refresh token for a user.
* `verify` checks a token's signature and expiry and returns its claims.
* `refresh_ttl_secs` reports the refresh token lifetime. It must equal `exp - iat` of the refresh tokens you create. authbox uses it to decide how long a "sessions revoked" note must be kept. See [Sessions and Revocation](sessions.md).

There is no `refresh` method. Rotation is handled by `AuthService::refresh_token`, which verifies and blacklists the old token and then calls `generate`.

## Built-in JWT manager

```rust
let tokens = DefaultJwtManager::new("a-secret-at-least-32-bytes-long!!", None, None)?;
```

* The first argument is your signing secret. Use a long random value, at least 32 bytes.
* The two optional arguments set the access token and refresh token lifetimes. Pass `None` to keep the defaults.
* `new` returns a `Result`, so a bad secret is caught at startup.

Load the secret from an environment variable or a secret manager. Never commit it.

## Token types

Every token carries a type, either access or refresh.

* `is_token_valid` accepts access tokens only.
* `refresh_token` and `logout` accept refresh tokens only.

Handing a refresh token to your middleware, or an access token to the refresh endpoint, fails with `InvalidToken`.

## Types

```rust
pub struct TokenPair {
    pub access_token: String,
    pub refresh_token: String,
    // plus expiry details
}
```

`Claims` is what `is_token_valid` returns. The fields you will use most:

| Field | Meaning |
|---|---|
| `sub` | The user id |
| `iat` | Issued at, in unix seconds |
| `typ` | The token type (access or refresh) |
| `jti()` | Unique token id, used for blacklisting |
| `exp()` | Expiry time in unix seconds |

`jti()` and `exp()` come from the `BlacklistableClaims` trait:

```rust
pub trait BlacklistableClaims {
    fn jti(&self) -> &str;
    fn exp(&self) -> i64;
}
```

## Using a custom token manager

```rust
pub struct MyTokenManager;

#[async_trait]
impl TokenManager for MyTokenManager {
    async fn generate(&self, user_id: &str) -> Result<TokenPair, BoxError> { todo!() }
    async fn verify(&self, token: &str) -> Result<Claims, BoxError> { todo!() }
    fn refresh_ttl_secs(&self) -> u64 { 60 * 60 * 24 }
}
```

Your tokens must include a unique `jti`, an `iat` in seconds, an `exp` and the token type, because blacklisting and session revocation depend on them.

## Summary

* Use `DefaultJwtManager` for standard JWTs.
* Implement `TokenManager` for any other token format.
* Keep `refresh_ttl_secs` in sync with your refresh token lifetime.
