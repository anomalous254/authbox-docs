# Configuring Token Blacklisting

Refresh token rotation requires a blacklist store.

The blacklist prevents reuse of previously rotated refresh tokens, ensuring that old refresh tokens cannot be reused after rotation.

---

## Why It Exists

When a refresh token is exchanged:

1. The current token is verified.
2. The current token is blacklisted.
3. A new token pair (access + refresh) is generated.

The original token can never be used again, improving security against token replay attacks.


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

- `is_blacklisted(jti)` checks if a token ID has already been used.
- `blacklist_token(jti, expires_at)` adds a token to the blacklist.


## Recommended Backends

**Production:**  

- Redis  
- SQL/NoSQL databases  

**Testing / Development:**  

- In-memory storage using a `HashSet`  


## Example: In-Memory Blacklist Store

```rust
use async_trait::async_trait;
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct MemoryBlacklistStore {
    pub tokens: Arc<Mutex<HashSet<String>>>,
}

impl MemoryBlacklistStore {
    pub fn new() -> Self {
        Self {
            tokens: Arc::new(Mutex::new(HashSet::new())),
        }
    }
}

#[derive(Debug)]
pub struct BlacklistError;

#[async_trait]
impl TokenBlacklistStore for MemoryBlacklistStore {
    type Error = BlacklistError;

    async fn is_blacklisted(&self, jti: &str) -> Result<bool, Self::Error> {
        let store = self.tokens.lock().unwrap();
        Ok(store.contains(jti))
    }

    async fn blacklist_token(&self, jti: &str, _expires_at: i64) -> Result<bool, Self::Error> {
        let mut store = self.tokens.lock().unwrap();
        store.insert(jti.to_string());
        Ok(true)
    }
}
```

- The in-memory store is suitable for testing or small-scale apps.  
- In production, use a persistent and distributed store like Redis to share blacklist data across multiple servers.

---

## Summary

- Token blacklisting ensures **rotated refresh tokens cannot be reused**.  
- Implement `TokenBlacklistStore` for custom storage.  
- Built-in in-memory implementation is provided for quick testing.  
- Production apps should use a distributed persistent store like Redis.
