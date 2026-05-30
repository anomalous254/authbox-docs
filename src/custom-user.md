# Custom User Types

AuthBox is fully generic over user models, allowing you to define your own domain-specific user structure.

---

## AuthUser Trait

All user types must implement the `AuthUser` trait. This ensures AuthBox can:

- Identify users
- Perform authentication lookups
- Manage passwords
- Track email verification status

```rust
/// Full Auth user model contract
pub trait AuthUser {
    fn id(&self) -> String;
    fn email(&self) -> &str;
    fn password_hash(&self) -> &str;
    fn is_email_verified(&self) -> bool;
    fn set_email_verified(&mut self, verified: bool);
    fn set_password_hash(&mut self, hash: String);
}
```


## Create Your User Type

Your user model must implement the `AuthUser` trait. You can also add custom fields as needed.

```rust
#[derive(Clone, Debug)]
pub struct YourUserModelType {
    // Required fields
    pub id: String,
    pub email: String,
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
}
```

## Custom Registration DTO

You can define your own registration input structure to match your application's needs.


### RegisterUserInput Trait

AuthBox only requires two core fields for registration:

- Email
- Password

These are accessed through the `RegisterUserInput` trait.

```rust
pub trait RegisterUserInput {
    fn email(&self) -> &str;
    fn password(&self) -> &str;
}
```


### Example DTO

You can extend the registration payload with any custom fields you need.

```rust
#[derive(Clone, Debug)]
pub struct SignUpUserPayload {
    // Required fields
    pub email: String,
    pub password: String,

    // Custom fields
    pub username: Option<String>,
    pub phone: Option<String>,
    pub country: Option<String>,
    pub city: Option<String>,
    pub age: Option<u32>,
}
```


### Implement RegisterUserInput

Only the required fields are exposed to AuthBox.

```rust
impl RegisterUserInput for SignUpUserPayload {
    fn email(&self) -> &str {
        &self.email
    }

    fn password(&self) -> &str {
        &self.password
    }
}
```


## Key Points

- AuthBox only depends on **email** and **password**
- You are free to add any extra fields for your domain
- Extra fields are passed through to your `UserStore` implementation
- Keeps authentication core simple and flexible

## Key Benefits

By implementing your own user type, you fully control:

- User structure
- Registration fields
- Domain-specific metadata

AuthBox only requires minimal traits to function, making it highly flexible.
