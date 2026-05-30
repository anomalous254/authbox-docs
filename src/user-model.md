# Creating a User Type

AuthBox stores and authenticates users through the `AuthUser` trait.

Your user model can contain any fields required by your application.

---

# Required Fields

AuthBox needs access to:

- User ID
- Email
- Password Hash
- Email Verification Status


# Example User

```rust
#[derive(Clone, Debug)]
pub struct User {
    // Required fields
    pub id: String,
    pub email: String,
    pub password_hash: String,
    pub is_email_verified: bool,
    
    // Your Custom fields
    pub username: Option<String>,
    pub phone: Option<String>,
    ...
}
```


# Implement AuthUser

> Your `User`  type must Implement the `AuthUser` trait

```rust
impl AuthUser for User {
    fn id(&self) -> String {
        self.id.clone()
    }

    fn email(&self) -> &str {
        &self.email
    }

    fn password_hash(&self) -> &str {
        &self.password_hash
    }

    fn is_email_verified(&self) -> bool {
        self.is_email_verified
    }

    fn set_email_verified(
        &mut self,
        verified: bool,
    ) {
        self.is_email_verified = verified;
    }

    fn set_password_hash(
        &mut self,
        hash: String,
    ) {
        self.password_hash = hash;
    }
}
```


# Why Email Verification Matters

The login flow checks:

```rust
store.is_email_verified(...)
```

before generating tokens.

Unverified users cannot authenticate.
