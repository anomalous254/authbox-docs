# Custom Storage Backends

authbox keeps all persistence behind the `UserStore` trait. That means you can use any database or storage system.

---

## Supported backends

* PostgreSQL
* MySQL
* SQLite
* MongoDB
* Redis
* REST APIs
* gRPC services
* In-memory storage (tests)

## The trait

```rust
#[async_trait]
pub trait UserStore: Send + Sync {
    async fn find_by_id(&self, user_id: &str) -> Result<Option<Box<dyn AuthUser>>, BoxError>;
    async fn find_by_email(&self, email: &str) -> Result<Option<Box<dyn AuthUser>>, BoxError>;
    async fn create_user(
        &self,
        input: &dyn RegisterUserInput,
        pass_hash: String,
    ) -> Result<Box<dyn AuthUser>, BoxError>;
    async fn update_user(&self, user: Box<dyn AuthUser>) -> Result<Box<dyn AuthUser>, BoxError>;
    async fn delete_user(&self, user_id: &str) -> Result<(), BoxError>;
    async fn check_email_verified(&self, user_id: &str) -> Result<bool, BoxError>;
}
```

The full method by method guide, with an example, is in [Implementing a User Store](user-store.md).

## Guidelines

* Make the store thread safe (`Arc`, connection pools and so on).
* Avoid blocking calls inside async methods.
* Return `Ok(None)` for a missing user, and keep `Err` for real failures.
* Put a UNIQUE index on the normalized email column.
* Report a unique violation from `create_user` as `AuthError::EmailAlreadyExists`.
* In `update_user`, merge the core fields into the stored record instead of replacing it, so your custom fields survive.
* Make `delete_user` succeed when the user is already gone.

## Other storage in authbox

The user store is not the only pluggable storage. Each of these can also be replaced:

| Trait | Default | Chapter |
|---|---|---|
| `TokenBlacklistStore` | `RedisBlacklistStore` | [Token blacklisting](token-blacklist.md) |
| `OneTimeTokenStore` | `RedisOttStore` | [One-time tokens](ott-store.md) |
| `UserCache` | `DefaultMemoryUserCache` | [User cache](user-cache.md) |

## Example use cases

* SQL backed authentication
* Multi-tenant user storage
* A microservice that fronts your user database
