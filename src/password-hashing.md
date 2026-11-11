# Configuring Password Hashing

authbox hashes passwords through the `PasswordHasher` trait. Passwords are never stored in plain text. They are hashed on registration, on reset and on change, and checked against the stored hash on login.

---

## The trait

```rust
pub trait PasswordHasher: Send + Sync {
    fn hash(&self, password: &str) -> String;
    fn verify(&self, password: &str, hash: &str) -> bool;
}
```

The methods are synchronous. authbox runs them on Tokio's blocking thread pool, so a slow hash never blocks your async runtime.

## Built-in hasher

```rust
let hasher = DefaultHasher;
```

`DefaultHasher` uses Argon2. It creates a salt for every password and handles hashing and verification for you.

## Password length

authbox accepts passwords between 12 and 128 characters. This is checked in `register`, `reset_password` and `change_password` before any hashing happens. The upper limit keeps very long inputs from turning into a denial of service.

## Login timing

When the email is unknown, `login` still verifies the password against a dummy hash created at startup. Every login attempt costs one hash verification, so timing does not reveal which emails are registered.

## Using a custom hasher

```rust
pub struct MyHasher;

impl PasswordHasher for MyHasher {
    fn hash(&self, password: &str) -> String {
        // your hashing logic
    }

    fn verify(&self, password: &str, hash: &str) -> bool {
        // your verification logic
    }
}
```

Pass it to the builder with `.hasher(MyHasher)`. Use a slow, salted algorithm such as Argon2, scrypt or bcrypt.

## Summary

* Use `DefaultHasher` for a secure default.
* Implement `PasswordHasher` for another algorithm.
* Hashing runs on blocking threads and never stalls async code.
