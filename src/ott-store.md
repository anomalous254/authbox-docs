# Configuring One-Time Tokens

authbox uses a one-time token (OTT) store for the short lived tokens in email verification and password reset. Tokens expire after a set time and can only be used once.

---

## Where they are used

* Email verification (valid for 24 hours)
* Password reset (valid for 15 minutes)

## Built-in RedisOttStore

```rust
let client = redis::Client::open("redis://127.0.0.1/")?;

let ott_store = RedisOttStore::new(client);
```

Unlike the blacklist store, `RedisOttStore::new` is not async.

Why Redis:

* Native TTL support
* Tokens expire automatically
* Shared across application instances
* No manual cleanup

## The trait

```rust
#[async_trait]
pub trait OneTimeTokenStore: Send + Sync {
    async fn set(&self, key: &str, value: &str, ttl_seconds: u64) -> Result<(), BoxError>;
    async fn get(&self, key: &str) -> Result<Option<String>, BoxError>;
    async fn delete(&self, key: &str) -> Result<(), BoxError>;
    async fn take(&self, key: &str) -> Result<Option<String>, BoxError>;
}
```

**`set`** stores a value with a time to live.

**`get`** reads a value without removing it.

**`delete`** removes a value.

**`take`** reads and deletes a value in one atomic step. This is the important one. `verify_email` and `reset_password` use it, so two simultaneous requests with the same token cannot both succeed. Only one of them gets `Some(..)`.

If you write your own store, `take` must be truly atomic. In Redis that means `GETDEL`, or a small Lua script. A separate `get` followed by `delete` is not safe.

## Key format

authbox prefixes every key with its purpose, like `verify:<token>` or `reset:<token>`. A verification token can never be used as a reset token, or the other way around.

## Failure handling

If a temporary failure happens after `verify_email` has claimed a token, authbox puts the token back with `set`, so the user can click the link again. If the token was spent for a permanent reason (the email is already verified, or the user no longer exists), it stays spent.

## Custom implementations

Implement `OneTimeTokenStore` to use KeyDB, Memcached, PostgreSQL, MongoDB or DynamoDB. An in-memory map works for tests only.

## Summary

* The OTT store holds verification and reset tokens.
* `take` makes each token single use.
* `RedisOttStore` is the recommended production choice.
