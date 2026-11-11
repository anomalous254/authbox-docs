# Implementing a User Store

`UserStore` is the one component you always write yourself. authbox never talks to your database directly. It calls this trait, so you can use any database, ORM or driver.

---

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

There are no associated types. Methods return `Box<dyn AuthUser>` and errors are a plain `BoxError`, so your store can use `sqlx::Error`, `diesel` errors or your own type and convert with `?` or `.into()`.

## Methods

**`find_by_id`** returns `Ok(Some(user))` when found and `Ok(None)` when the user does not exist. Reserve `Err` for real backend failures.

**`find_by_email`** works the same way. authbox passes an email that is already trimmed and lowercased.

**`create_user`** receives the registration input and the already hashed password. Read your custom fields by downcasting the input with `as_any` (see [Creating a Registration DTO](registration-dto.md)).

**`update_user`** receives the changed user. authbox uses it to set the verified flag and to change the password hash. It only has the core `AuthUser` fields, so merge those into your stored record instead of replacing the whole record. Otherwise custom fields like `username` could be lost.

**`delete_user`** removes the user. Deleting a user that is already gone should still succeed, because authbox also calls it to roll back a failed registration.

**`check_email_verified`** returns whether the user has verified their email.

## Unique emails

The `find_by_email` check inside `register` is only a fast path. Two signups at the same moment can both pass it. To make duplicates impossible:

1. Put a UNIQUE index on the normalized email column in your database.
2. Make `create_user` report a unique violation as `AuthError::EmailAlreadyExists`.

authbox maps store errors in `map_store_create_err`. By default it recognizes an `AuthError::EmailAlreadyExists` wrapped in the `BoxError`. So the simplest approach is to return that from your store:

```rust
// inside create_user, for sqlx + Postgres
if let Err(e) = result {
    if let Some(db) = e.as_database_error() {
        if db.is_unique_violation() {
            return Err(Box::new(AuthError::EmailAlreadyExists));
        }
    }
    return Err(Box::new(e));
}
```

Store the normalized email (trimmed and lowercased) yourself. authbox only normalizes emails for lookups.

## Example: in-memory store

This is the store used in the test fixtures. It is useful for tests and for learning the trait.

```rust
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

#[derive(Clone, Default)]
pub struct TestStore {
    pub users: Arc<Mutex<HashMap<String, TestUser>>>,
}

#[async_trait]
impl UserStore for TestStore {
    async fn find_by_id(&self, user_id: &str) -> Result<Option<Box<dyn AuthUser>>, BoxError> {
        Ok(self.users.lock().unwrap().get(user_id).cloned().map(boxed))
    }

    async fn find_by_email(&self, email: &str) -> Result<Option<Box<dyn AuthUser>>, BoxError> {
        let users = self.users.lock().unwrap();
        Ok(users.values().find(|u| u.email == email).cloned().map(boxed))
    }

    async fn create_user(
        &self,
        input: &dyn RegisterUserInput,
        pass_hash: String,
    ) -> Result<Box<dyn AuthUser>, BoxError> {
        let email = input.email().trim().to_lowercase();
        let dto = input.as_any().downcast_ref::<RegisterDto>();

        // duplicate check and insert share one lock, like a UNIQUE index would
        let mut users = self.users.lock().unwrap();
        if users.values().any(|u| u.email == email) {
            return Err("unique violation".into());
        }

        let user = TestUser {
            id: Uuid::new_v4().to_string(),
            email,
            password_hash: pass_hash,
            is_email_verified: false,
            username: dto.and_then(|d| d.username.clone()),
            phone: dto.and_then(|d| d.phone.clone()),
            country: dto.and_then(|d| d.country.clone()),
            city: dto.and_then(|d| d.city.clone()),
            age: dto.and_then(|d| d.age),
        };

        users.insert(user.id.clone(), user.clone());
        Ok(boxed(user))
    }

    async fn update_user(&self, user: Box<dyn AuthUser>) -> Result<Box<dyn AuthUser>, BoxError> {
        let mut users = self.users.lock().unwrap();
        let existing = users.get_mut(&user.id()).ok_or("user not found")?;

        // merge the core fields so username, phone etc. survive
        existing.email = user.email().to_owned();
        existing.password_hash = user.password_hash().to_owned();
        existing.is_email_verified = user.is_email_verified();

        Ok(boxed(existing.clone()))
    }

    async fn delete_user(&self, user_id: &str) -> Result<(), BoxError> {
        self.users.lock().unwrap().remove(user_id);
        Ok(())
    }

    async fn check_email_verified(&self, user_id: &str) -> Result<bool, BoxError> {
        match self.users.lock().unwrap().get(user_id) {
            Some(user) => Ok(user.is_email_verified),
            None => Err("user not found".into()),
        }
    }
}

fn boxed(u: TestUser) -> Box<dyn AuthUser> {
    Box::new(u)
}
```

Clones of `TestStore` share the same map. A test can hand one copy to the builder and keep another to inspect the data afterwards.

## Notes

* Make the store thread safe. Use `Arc`, connection pools or similar.
* Do not block inside async methods.
* Index the email column so lookups stay fast.
* Password hashing is done by authbox before `create_user` is called.
