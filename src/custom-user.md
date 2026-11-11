# Custom User Types

authbox works with any user model you define, as long as it implements `AuthUser`. It does not care about the rest of your fields.

---

## AuthUser

```rust
pub trait AuthUser: Send + Sync {
    fn id(&self) -> String;
    fn email(&self) -> &str;
    fn password_hash(&self) -> &str;
    fn is_email_verified(&self) -> bool;
    fn set_email_verified(&mut self, verified: bool);
    fn set_password_hash(&mut self, hash: String);

    /// Extra public fields for the `me()` response.
    fn extra(&self) -> Option<serde_json::Value> {
        None
    }
}
```

This is how authbox can:

* identify users
* check passwords
* change passwords
* track email verification
* include your public fields in `me()`

## Create your user type

```rust
#[derive(Clone, Debug, Serialize)]
pub struct YourUserModelType {
    // Required fields
    pub id: String,
    pub email: String,
    #[serde(skip_serializing)]
    pub password_hash: String,
    pub is_email_verified: bool,

    // Custom fields
    pub username: Option<String>,
    pub phone: Option<String>,
    pub country: Option<String>,
    pub city: Option<String>,
    pub age: Option<u32>,
}

impl AuthUser for YourUserModelType {
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

    fn set_email_verified(&mut self, verified: bool) {
        self.is_email_verified = verified;
    }

    fn set_password_hash(&mut self, hash: String) {
        self.password_hash = hash;
    }

    fn extra(&self) -> Option<serde_json::Value> {
        serde_json::to_value(self).ok()
    }
}
```

`#[serde(skip_serializing)]` on the hash keeps it out of `extra()`, and so out of `me()` and the user cache.

## Custom registration DTO

AuthBox only needs an email and a password at signup. Everything else is yours.

```rust
pub trait RegisterUserInput: Send + Sync {
    fn email(&self) -> &str;
    fn password(&self) -> &str;
    fn as_any(&self) -> &dyn Any;
}
```

Implement it for your DTO, and downcast with `as_any` inside your `UserStore::create_user` to read the custom fields. A full example is in [Creating a Registration DTO](registration-dto.md).

## Key points

* authbox depends only on email and password for registration.
* Add any extra fields you need.
* Extra fields reach your `UserStore`, where you save them.
* `update_user` only carries the core fields, so merge them into the stored record.
* Never put secrets in `extra()`. It is returned to clients and cached.
