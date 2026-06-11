# Configuring One-Time Token Storage

AuthBox uses a one-time token store for temporary authentication tokens used in verification and recovery workflows.

These tokens are short-lived and automatically expire after a configured TTL.

---

## Used By

One-time tokens are used for:

- Email verification
- Password reset
- Magic links
- One-time authentication flows

---

## Built-in RedisOttStore

AuthBox ships with a built-in Redis implementation of `OneTimeTokenStore`.

```rust
let client = redis::Client::open("redis://127.0.0.1/")?;

let ott_store = RedisOttStore::new(client);
```

### Why Redis?

- Production-ready
- Native TTL support
- Automatic token expiration
- Works across multiple application instances
- No manual cleanup required

AuthBox stores one-time tokens in Redis with an expiration time, and Redis automatically removes them when the TTL expires.

---

## OneTimeTokenStore Trait

```rust
use async_trait::async_trait;

#[async_trait]
pub trait OneTimeTokenStore {
    type Error;

    async fn set(
        &self,
        key: &str,
        value: &str,
        ttl_seconds: u64,
    ) -> Result<(), Self::Error>;

    async fn get(
        &self,
        key: &str,
    ) -> Result<Option<String>, Self::Error>;

    async fn delete(
        &self,
        key: &str,
    ) -> Result<(), Self::Error>;
}
```

---

## Methods

### `set`

Stores a token value with a time-to-live.

```rust
store
    .set(
        key,
        value,
        ttl_seconds,
    )
    .await?;
```

### `get`

Retrieves a stored token.

```rust
let value = store.get(key).await?;
```

### `delete`

Removes a token from storage.

```rust
store.delete(key).await?;
```

---

## Token Lifetimes

AuthBox uses different expiration periods depending on the workflow.

### Email Verification

```rust
60 * 60 * 24
```

24 hours.

### Password Reset

```rust
60 * 15
```

15 minutes.

You may configure different expiration times depending on your application's requirements.

---

## Default Redis Implementation

AuthBox includes the following Redis-backed implementation:

```rust
use async_trait::async_trait;
use authbox_core::traits::OneTimeTokenStore;
use redis::AsyncCommands;

#[derive(Clone)]
pub struct RedisOttStore {
    pub client: redis::Client,
}

impl RedisOttStore {
    pub fn new(client: redis::Client) -> Self {
        Self { client }
    }
}

#[async_trait]
impl OneTimeTokenStore for RedisOttStore {
    type Error = redis::RedisError;

    async fn set(
        &self,
        key: &str,
        value: &str,
        ttl_seconds: u64,
    ) -> Result<(), Self::Error> {
        let mut conn = self
            .client
            .get_multiplexed_async_connection()
            .await?;

        conn.set_ex::<_, _, ()>(
            key,
            value,
            ttl_seconds,
        )
        .await?;

        Ok(())
    }

    async fn get(
        &self,
        key: &str,
    ) -> Result<Option<String>, Self::Error> {
        let mut conn = self
            .client
            .get_multiplexed_async_connection()
            .await?;

        let value: Option<String> =
            conn.get(key).await?;

        Ok(value)
    }

    async fn delete(
        &self,
        key: &str,
    ) -> Result<(), Self::Error> {
        let mut conn = self
            .client
            .get_multiplexed_async_connection()
            .await?;

        conn.del::<_, ()>(key).await?;

        Ok(())
    }
}
```

---

## Using RedisOttStore

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
    let client = redis::Client::open(
        "redis://127.0.0.1/"
    )
    .expect("failed to connect to redis");

    AuthService::builder()
        .store(TestStore::new())
        .hasher(DefaultHasher)
        .tokens(DefaultJwtManager::new("secret"))
        .blacklist(
            RedisBlacklistStore::new(
                client.clone(),
            ),
        )
        .email_sender(MockEmailSender)
        .email_templates(MockTemplates)
        .ott_store(
            RedisOttStore::new(client),
        )
        .build()
}
```

---

## Custom Implementations

You can provide your own token storage backend by implementing `OneTimeTokenStore`.

Common choices include:

- Redis
- KeyDB
- Memcached
- PostgreSQL
- MySQL
- MongoDB
- DynamoDB
- In-memory `HashMap` (testing only)

---

## Summary

- AuthBox requires a `OneTimeTokenStore` for temporary authentication tokens.
- AuthBox includes a built-in `RedisOttStore`.
- Tokens automatically expire using Redis TTL support.
- Used for email verification, password reset, magic links, and similar flows.
- Custom storage backends can be implemented through `OneTimeTokenStore`.
- Redis is the recommended choice for production deployments.