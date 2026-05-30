# Configuring Password Hashing

AuthBox handles password hashing through the `PasswordHasher` trait.

Passwords are **never stored in plaintext**. During registration, AuthBox hashes the user's password before persisting it, and during login it verifies the provided password against the stored hash.

---

## PasswordHasher Trait

To support different hashing algorithms, AuthBox defines the following trait:

```rust
pub trait PasswordHasher {
    fn hash(&self, password: &str) -> String;
    fn verify(&self, password: &str, hash: &str) -> bool;
}
```

- `hash` generates a secure hash of the password.
- `verify` checks a password against a stored hash.


## Built-in Hasher

AuthBox ships with a default Argon2 implementation.

```rust
let hasher = DefaultHasher;
```

- This is ready to use out of the box.
- Handles salt generation, hashing, and verification.


## Registration Flow

During user registration:

```rust
let hash = self.hasher.hash(input.password());
```

The resulting `hash` is stored in your `UserStore` instead of the plaintext password.


 ## Login Flow

During login:

```rust
let valid = self.hasher.verify(password, user.password_hash());
```

Authentication only proceeds if `valid` is `true`.


## Using a Custom Hasher

You can implement your own password hashing strategy by implementing the `PasswordHasher` trait:

```rust
pub struct MyHasher;

impl PasswordHasher for MyHasher {
    fn hash(&self, password: &str) -> String {
        // custom hashing logic
    }

    fn verify(&self, password: &str, hash: &str) -> bool {
        // custom verification logic
    }
}
```

You can then configure AuthBox to use your hasher instead of the default.

---

## Summary

- **Default option**: Use `DefaultHasher` (Argon2) — simple and secure.
- **Custom option**: Implement your own `PasswordHasher` for Bcrypt, Scrypt, HSM-backed hashing, or any custom algorithm.

AuthBox works with **any** implementation of `PasswordHasher`, giving you flexibility without changing the core authentication logic.
