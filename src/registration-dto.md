# Creating a Registration DTO

AuthBox does not force a specific registration payload.

Instead, applications provide their own registration DTO and implement the `RegisterUserInput` trait.

This allows your application to collect any additional information during registration while still providing the fields required by AuthBox.

---

# Required Fields

AuthBox only requires:

- Email
- Password

These fields are accessed through the `RegisterUserInput` trait.


# Example DTO

```rust
#[derive(Clone, Debug)]
pub struct SignUpUserPayload {
    // Required fileds
    pub email: String,
    pub password: String,
    
    // Your Custom fileds
    pub username: Option<String>,
    pub phone: Option<String>,
    pub country: Option<String>,
    ...
}
```


# Implement RegisterUserInput

```rust
impl RegisterUserInput for SignUpUserPayload  {
    fn email(&self) -> &str {
        &self.email
    }

    fn password(&self) -> &str {
        &self.password
    }
}
```


# How AuthBox Uses It

During registration:

```rust
auth.register(dto).await?;
```

AuthBox reads:

```rust
input.email()
input.password()
```

and passes the DTO to your `UserStore`.

This allows you to persist both authentication fields and application-specific fields.
