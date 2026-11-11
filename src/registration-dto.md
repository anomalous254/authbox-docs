# Creating a Registration DTO

authbox does not force a registration payload on you. You define your own DTO and implement the `RegisterUserInput` trait for it. That lets you collect any extra information at signup while still giving authbox the two fields it needs.

---

## Required fields

authbox only needs:

* Email
* Password

## The trait

```rust
pub trait RegisterUserInput: Send + Sync {
    fn email(&self) -> &str;
    fn password(&self) -> &str;
    fn as_any(&self) -> &dyn Any;
}
```

`as_any` is what lets your own `UserStore` get the full DTO back, including your custom fields.

## Example DTO

```rust
use serde::Deserialize;
use std::any::Any;

#[derive(Clone, Debug, Deserialize)]
pub struct RegisterDto {
    // Required fields
    pub email: String,
    pub password: String,

    // Your custom fields
    pub username: Option<String>,
    pub phone: Option<String>,
    pub country: Option<String>,
    pub city: Option<String>,
    pub age: Option<u32>,
}

impl RegisterUserInput for RegisterDto {
    fn email(&self) -> &str {
        &self.email
    }

    fn password(&self) -> &str {
        &self.password
    }

    fn as_any(&self) -> &dyn Any {
        self
    }
}
```

## How authbox uses it

You pass a reference to `register`:

```rust
auth.register(&dto).await?;
```

authbox reads `email()` and `password()`, then hands the same input to your `UserStore::create_user`. There you downcast it to read the custom fields:

```rust
let dto = input.as_any().downcast_ref::<RegisterDto>();

let username = dto.and_then(|d| d.username.clone());
```

If you want a hard failure when the wrong type arrives, use `ok_or`:

```rust
let dto = input
    .as_any()
    .downcast_ref::<RegisterDto>()
    .ok_or("wrong dto")?;
```

## Notes

* The password length (12 to 128 characters) is checked by authbox before anything is created.
* The email is trimmed and lowercased by authbox for lookups. Save the normalized email in your store as well. See [Implementing a User Store](user-store.md).
