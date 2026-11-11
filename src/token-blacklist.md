# Configuring Token Blacklisting

The blacklist is what makes revocation work. It does two jobs:

1. **Single token revocation.** A rotated or logged out refresh token can never be used again.
2. **Session revocation.** Every token a user has issued up to a certain moment can be voided at once.

---

## How rotation uses it

When a refresh token is exchanged:

1. The token is verified, and checked against the blacklist.
2. The token is blacklisted.
3. A new token pair is generated.

`blacklist_token` reports whether the token was newly blacklisted. If it was already there, another request got to it first and the exchange fails with `BlacklistedToken`. This stops two requests from rotating the same token.

## Built-in RedisBlacklistStore

```rust
let client = redis::Client::open("redis://127.0.0.1/")?;

let blacklist = RedisBlacklistStore::new(client).await?;
```

Note the `.await?`. The Redis store is created asynchronously.

Why Redis:

* Production ready
* Entries expire on their own through Redis TTL
* Shared across all your application instances
* No cleanup jobs

## The trait

```rust
#[async_trait]
pub trait TokenBlacklistStore: Send + Sync {
    async fn is_blacklisted(&self, jti: &str) -> Result<bool, BoxError>;
    async fn blacklist_token(&self, jti: &str, expires_at: i64) -> Result<bool, BoxError>;

    /// "Every token for this user issued at or before `cutoff` is void."
    /// The note can be deleted after `ttl_seconds`.
    async fn revoke_user_sessions(
        &self,
        user_id: &str,
        cutoff: i64,
        ttl_seconds: u64,
    ) -> Result<(), BoxError>;

    /// Read the note back. None means no sessions have been revoked.
    async fn sessions_revoked_before(&self, user_id: &str) -> Result<Option<i64>, BoxError>;
}
```

### Methods

**`is_blacklisted(jti)`** returns whether a token id has been blacklisted.

**`blacklist_token(jti, expires_at)`** blacklists a token until it would have expired anyway. Return `true` when the token was newly blacklisted and `false` when it was already there.

**`revoke_user_sessions(user_id, cutoff, ttl_seconds)`** stores a cutoff time for a user. authbox passes the current time in unix seconds as `cutoff`, and the refresh token lifetime as `ttl_seconds`. After that long, every older token has expired on its own, so the note can be dropped.

**`sessions_revoked_before(user_id)`** returns the stored cutoff, or `None` when there is none.

## How checks work

Every call to `is_token_valid` and `refresh_token` runs these checks:

1. The signature and expiry are valid.
2. The token type is the expected one.
3. The token id is not blacklisted.
4. The token was not issued at or before the user's session cutoff.

## Custom implementations

Implement `TokenBlacklistStore` to use another backend, such as PostgreSQL, MySQL, MongoDB or DynamoDB. An in-memory map is fine for tests, but it will not be shared between instances.

Make sure `blacklist_token` is atomic. Two simultaneous calls for the same token must not both return `true`.

## Summary

* Rotation and logout depend on the blacklist.
* Session revocation uses the same trait.
* Use `RedisBlacklistStore` in production.
