# Custom Storage Backends

AuthBox uses the `UserStore` trait to abstract all persistence logic.

This allows integration with any database or storage system.

---

## Supported Backends

You can implement storage using:

- PostgreSQL
- MySQL
- SQLite
- MongoDB
- Redis
- REST APIs
- gRPC services
- In-memory storage

## UserStore Trait

```rust
#[async_trait]
pub trait UserStore {
    type Error;
    type User: AuthUser;
    type RegisterDto: RegisterUserInput;

    async fn find_by_id(&self, user_id: &str) -> Option<Self::User>;

    async fn find_by_email(&self, email: &str) -> Option<Self::User>;

    async fn create_user(
        &self,
        input: Self::RegisterDto,
        password_hash: String,
    ) -> Result<Self::User, Self::Error>;

    async fn update_user(
        &self,
        user: Self::User,
    ) -> Result<Self::User, Self::Error>;

    async fn delete_user(
        &self,
        user_id: &str,
    ) -> Result<(), Self::Error>;

    async fn is_email_verified(
        &self,
        user_id: &str,
    ) -> Result<bool, Self::Error>;
}
```


## Implementation Guidelines

When implementing a custom store:

- Ensure thread safety (`Arc`, connection pools, etc.)
- Avoid blocking operations inside async methods
- Handle errors using your backend’s error type
- Keep queries efficient and indexed
- Ensure email lookup is fast (commonly indexed)


# Example Use Cases

- SQL-backed authentication system
- Multi-tenant user storage
- Microservice-based auth backend
- High-performance cache layer (Redis)


## Key Benefit

AuthBox does not dictate your database.

You control persistence entirely.
