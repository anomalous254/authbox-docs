# Configuring Token Blacklisting

Refresh token rotation requires a blacklist store.

The blacklist ensures that **rotated refresh tokens cannot be reused**, preventing token replay attacks.

---

## How It Works

When a refresh token is exchanged:

1. The current token is verified.
2. The current token is blacklisted.
3. A new token pair (access + refresh) is generated.

The original token is invalidated immediately and can never be used again.

---

## Built-in RedisBlacklistStore

AuthBox ships with a built-in Redis implementation of `TokenBlacklistStore`.

```rust
let client = redis::Client::open("redis://127.0.0.1/")?;

let blacklist = RedisBlacklistStore::new(client);
```

### Why Redis?

- Production-ready
- Automatically expires blacklisted tokens using Redis TTL
- Works across multiple application instances
- No manual cleanup required

The blacklist entry is stored until the refresh token's expiration time, after which Redis automatically removes it.

---

## TokenBlacklistStore Trait

```rust
use async_trait::async_trait;

#[async_trait]
pub trait TokenBlacklistStore {
    type Error;

    async fn is_blacklisted(
        &self,
        jti: &str,
    ) -> Result<bool, Self::Error>;

    async fn blacklist_token(
        &self,
        jti: &str,
        expires_at: i64,
    ) -> Result<bool, Self::Error>;
}
```

### Methods

#### `is_blacklisted(jti)`

Checks whether a token identifier has already been blacklisted.

```rust
let blacklisted = store.is_blacklisted(jti).await?;
```

#### `blacklist_token(jti, expires_at)`

Adds a token identifier to the blacklist until its expiration time.

```rust
store
    .blacklist_token(jti, expires_at)
    .await?;
```

---

## Default Redis Implementation

AuthBox includes the following implementation:

```rust
use async_trait::async_trait;
use authbox_core::traits::TokenBlacklistStore;
use redis::AsyncCommands;
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Clone)]
pub struct RedisBlacklistStore {
    pub client: redis::Client,
}

impl RedisBlacklistStore {
    pub fn new(client: redis::Client) -> Self {
        Self { client }
    }
}

#[derive(Debug)]
pub enum RedisBlacklistError {
    Redis(redis::RedisError),
}

#[async_trait]
impl TokenBlacklistStore for RedisBlacklistStore {
    type Error = RedisBlacklistError;

    async fn is_blacklisted(&self, jti: &str) -> Result<bool, Self::Error> {
        let mut conn = self
            .client
            .get_multiplexed_async_connection()
            .await
            .map_err(RedisBlacklistError::Redis)?;

        let exists: bool = conn
            .exists(jti)
            .await
            .map_err(RedisBlacklistError::Redis)?;

        Ok(exists)
    }

    async fn blacklist_token(
        &self,
        jti: &str,
        expires_at: i64,
    ) -> Result<bool, Self::Error> {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        if expires_at <= now {
            return Ok(true);
        }

        let ttl = (expires_at - now) as u64;

        let mut conn = self
            .client
            .get_multiplexed_async_connection()
            .await
            .map_err(RedisBlacklistError::Redis)?;

        let _: () = conn
            .set_ex(jti, "1", ttl)
            .await
            .map_err(RedisBlacklistError::Redis)?;

        Ok(true)
    }
}
```

---

## Using RedisBlacklistStore

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

---

## Custom Implementations

You can provide your own blacklist backend by implementing `TokenBlacklistStore`.

Common choices include:

- Redis
- PostgreSQL
- MySQL
- MongoDB
- DynamoDB
- In-memory `HashSet` (testing only)

---

## Summary

- Refresh token rotation requires token blacklisting.
- AuthBox includes a built-in `RedisBlacklistStore`.
- Blacklisted tokens are automatically removed when they expire.
- Custom blacklist backends can be implemented through `TokenBlacklistStore`.
- Redis is the recommended choice for production deployments.