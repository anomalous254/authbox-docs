# Configuring One-Time Token Store

AuthBox uses the `OneTimeTokenStore` trait for temporary tokens used in authentication workflows.

These tokens are short-lived and are typically stored in Redis, memory, or another key-value store.

---

## Used By

One-time tokens are used for:

- Email verification
- Password reset
- Magic links
- One-time authentication flows


## OneTimeTokenStore Trait

To use one-time token features, you must provide an implementation of:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait OneTimeTokenStore {
    async fn set(
        &self,
        key: &str,
        value: &str,
        ttl_seconds: u64,
    );

    async fn get(
        &self,
        key: &str,
    ) -> Option<String>;

    async fn delete(
        &self,
        key: &str,
    );
}
```

## Methods

### `set`

Stores a token value with a time-to-live.

```rust
async fn set(
    &self,
    key: &str,
    value: &str,
    ttl_seconds: u64,
);
```

### `get`

Retrieves a stored token.

```rust
async fn get(
    &self,
    key: &str,
) -> Option<String>;
```

### `delete`

Removes a token from storage.

```rust
async fn delete(
    &self,
    key: &str,
);
```


## Token Lifetimes

AuthBox uses different expiration periods depending on the workflow.

## Email Verification

```rust
60 * 60 * 24
```

24 hours.

## Password Reset

```rust
60 * 15
```

15 minutes.

You may configure different expiration times depending on your application's requirements.


## Example: In-Memory Store

The following example demonstrates a simple in-memory implementation.

```rust
use async_trait::async_trait;
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};

#[derive(Clone)]
pub struct MemoryOttStore {
    pub store: Arc<Mutex<HashMap<String, String>>>,
}

impl MemoryOttStore {
    pub fn new() -> Self {
        Self {
            store: Arc::new(
                Mutex::new(HashMap::new())
            ),
        }
    }
}

#[async_trait]
impl OneTimeTokenStore for MemoryOttStore {
    async fn set(
        &self,
        key: &str,
        value: &str,
        _: u64,
    ) {
        let mut store =
            self.store.lock().unwrap();

        store.insert(
            key.to_string(),
            value.to_string(),
        );
    }

    async fn get(
        &self,
        key: &str,
    ) -> Option<String> {
        let store =
            self.store.lock().unwrap();

        store.get(key).cloned()
    }

    async fn delete(
        &self,
        key: &str,
    ) {
        let mut store =
            self.store.lock().unwrap();

        store.remove(key);
    }
}
```


## Recommended Backends

For production workloads:

- Redis
- KeyDB
- Memcached
- Distributed cache systems

For development and testing:

- In-memory `HashMap`
- Mock implementations

---

## Summary

- AuthBox requires a `OneTimeTokenStore` implementation for temporary authentication tokens.
- Used by email verification, password reset, magic links, and similar flows.
- You can use Redis, memory, or any custom storage backend.
- The provided in-memory implementation is useful for local development and testing.
- Production deployments should use a persistent or distributed cache with TTL support.
