# Configuring the User Cache

The user cache is optional. It speeds up `me()` by keeping user data in memory so repeated requests skip your database.

---

## What is cached

Only user data is cached. The access token is validated on every `me()` call, including the blacklist and session revocation checks, so a revoked token never gets data out of the cache.

The cached value is the serialized `Me` response (id, email, verification flag and your `extra()` data), stored under the user id.

## Built-in cache

The `authbox-cache` crate provides `DefaultMemoryUserCache`, an in-memory cache built on [moka](https://github.com/moka-rs/moka).

```rust
let auth = AuthService::builder()
    // ...other components
    .user_cache(DefaultMemoryUserCache::default())
    .build()?;
```

Leave out `.user_cache(...)` and caching is simply off.

## The trait

```rust
#[async_trait]
pub trait UserCache: Send + Sync {
    async fn get(&self, user_id: &str) -> Result<Option<String>, BoxError>;
    async fn set(
        &self,
        user_id: &str,
        value: &str,
        ttl_seconds: Option<u64>,
    ) -> Result<(), BoxError>;
    async fn invalidate(&self, user_id: &str) -> Result<(), BoxError>;
}
```

Values are plain JSON strings. authbox calls `set` with `ttl_seconds` as `None`, so your cache decides the default lifetime.

## When the cache is cleared

authbox invalidates a user's entry when:

* the email is verified
* the password is reset or changed
* the account is deleted
* a registration is rolled back

If you change user data yourself, for example updating a username in your own endpoint, call:

```rust
auth.invalidate_user_cache(&user_id).await?;
```

Otherwise the old data is served until the entry expires.

## Failures never break a request

* A failed read logs a warning and falls back to your `UserStore`.
* A cached value that cannot be parsed is treated as a miss and overwritten.
* A failed write logs a warning and the response is still returned.
* A failed invalidation after a password or account change logs a warning, and the entry expires with its TTL.

## Metrics

When a cache is configured, authbox records these counters through the `metrics` crate:

| Counter | Meaning |
|---|---|
| `authbox_cache_hits_total` | Served from the cache |
| `authbox_cache_misses_total` | Not in the cache |
| `authbox_cache_read_failures_total` | The cache read failed |
| `authbox_cache_write_failures_total` | The cache write failed |
| `authbox_cache_invalidate_failures_total` | The invalidation failed |

Install any `metrics` exporter in your app, such as Prometheus, to see them.

## Custom caches

Implement `UserCache` to back the cache with something else, such as Redis, so several instances share it.

```rust
pub struct MyCache { /* your client */ }

#[async_trait]
impl UserCache for MyCache {
    async fn get(&self, user_id: &str) -> Result<Option<String>, BoxError> { todo!() }

    async fn set(
        &self,
        user_id: &str,
        value: &str,
        ttl_seconds: Option<u64>,
    ) -> Result<(), BoxError> { todo!() }

    async fn invalidate(&self, user_id: &str) -> Result<(), BoxError> { todo!() }
}
```

> With the in-memory cache, each application instance has its own copy. After a change on one instance, other instances can serve old data until the entry expires. Use a shared cache, or a short TTL, if that matters for you.

## Summary

* The cache is optional and only affects `me()`.
* `DefaultMemoryUserCache` works out of the box.
* Implement `UserCache` for your own backend.
* Cache problems are logged and counted, never fatal.
