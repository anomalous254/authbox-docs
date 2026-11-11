# Creating a User Type

authbox works with users through the `AuthUser` trait. Your user type can hold whatever fields your app needs.

---

## Required data

authbox needs access to:

* User id
* Email
* Password hash
* Email verification status

## The trait

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

`extra` is optional. It has a default that returns `None`.

## Example user

```rust
use serde::Serialize;

#[derive(Clone, Debug, Serialize)]
pub struct User {
    // Required fields
    pub id: String,
    pub email: String,
    #[serde(skip_serializing)]
    pub password_hash: String,
    pub is_email_verified: bool,

    // Your custom fields
    pub username: Option<String>,
    pub phone: Option<String>,
}

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

    fn set_email_verified(&mut self, verified: bool) {
        self.is_email_verified = verified;
    }

    fn set_password_hash(&mut self, hash: String) {
        self.password_hash = hash;
    }

    // whole user as json, minus the hash (serde skips it)
    fn extra(&self) -> Option<serde_json::Value> {
        serde_json::to_value(self).ok()
    }
}
```

## The `extra` method

`me()` returns an id, an email, the verification flag and whatever `extra()` gives back. Use it to expose public profile fields such as a username or avatar.

Keep the password hash out of it. In the example, `#[serde(skip_serializing)]` makes sure the hash is never serialized. The `extra` value is also what gets stored in the user cache, so never put secrets in it.

## Why email verification matters

`login` checks `is_email_verified()` after the password check, and only verified users receive tokens.
