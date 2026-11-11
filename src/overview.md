# authbox overview

authbox is a lightweight, modular, async-first authentication framework for Rust.

It gives you a complete authentication workflow while staying independent of your database, cache, email service and web framework. Every major component is a trait, so you can use the included defaults or plug in your own.

At the center is `AuthService`. It handles registration, login, token refresh, email verification, password recovery, session revocation and account management.

---

# Core philosophy

## Modular

Every component is replaceable through a trait. Use PostgreSQL, MySQL, SQLite, MongoDB or anything else for users, and change it later without touching the authentication logic.

## Async first

All operations are async and built for Tokio. Password hashing runs on blocking threads, so slow Argon2 work never stalls the runtime.

## Framework agnostic

authbox does not depend on Axum, Actix, Rocket or Warp. It works in any Rust service.

---

# Building an AuthService

```rust
let auth = AuthService::builder()
    .store(user_store)
    .hasher(password_hasher)
    .tokens(token_manager)
    .blacklist(blacklist_store)
    .email_sender(email_provider)
    .email_templates(email_templates)
    .ott_store(one_time_token_store)
    .user_cache(user_cache)      // optional
    .build()?;
```

`build()` returns a `Result`, so a missing required component is reported at startup instead of at the first request.

`AuthService` is a plain, non generic struct. It holds its components behind an `Arc`, so `.clone()` is cheap and it is safe to put in your application state.

---

# Components

| Component | Responsibility | Default |
|---|---|---|
| `UserStore` | User persistence and lookup | None |
| `PasswordHasher` | Hash and verify passwords | `DefaultHasher` |
| `TokenManager` | Create and verify access and refresh tokens | `DefaultJwtManager` |
| `TokenBlacklistStore` | Token and session revocation | `RedisBlacklistStore` |
| `OneTimeTokenStore` | Short lived verification and reset tokens | `RedisOttStore` |
| `EmailProvider` | Send emails | Resend, SendGrid, SMTP |
| `EmailTemplateConfig` | Email subjects and bodies | None |
| `UserCache` (optional) | Cache user data for `me()` | `DefaultMemoryUserCache` |

---

# Authentication flow

```text
Register
   |
   v
Email verification
   |
   v
Login
   |
   v
Access token + refresh token
   |
   v
Refresh (rotation)
   |
   v
Logout, logout all, or session revocation
```

Password recovery is available at any point.

---

# AuthService API

Every method returns `Result<_, AuthError>`. See [Error Handling](errors.md) for the full list.

## register

```rust
pub async fn register(
    &self,
    input: &dyn RegisterUserInput,
) -> Result<Box<dyn AuthUser>, AuthError>
```

Creates a new account and sends a verification email.

1. Check the password length (12 to 128 characters).
2. Check whether the email already exists.
3. Hash the password.
4. Create the user through `UserStore`.
5. Store a verification token (valid for 24 hours).
6. Send the verification email.

If storing the token or sending the email fails, the user is deleted again so you never end up with an account that cannot be verified.

> The email check in step 2 is only a fast path. Two signups at the same moment can both pass it, so your database needs a UNIQUE index on the email column. See [Implementing a User Store](user-store.md).

## verify_email

```rust
pub async fn verify_email(&self, token: &str) -> Result<(), AuthError>
```

Marks the email as verified. The token is claimed with an atomic `take`, so two simultaneous requests cannot both use it. If a temporary failure happens after the claim, the token is put back so the user can try again.

## login

```rust
pub async fn login(&self, email: &str, password: &str) -> Result<TokenPair, AuthError>
```

1. Find the user by email.
2. Verify the password.
3. Check that the email is verified.
4. Generate a token pair.

A password hash is verified even for unknown emails, so response times do not reveal which accounts exist. The password is checked before the verification status is revealed.

## me

```rust
pub async fn me(&self, access_token: &str) -> Result<Me, AuthError>
```

Returns the current user. The token is always validated. Only the user data is cached. See [Configuring the User Cache](user-cache.md).

## refresh_token

```rust
pub async fn refresh_token(&self, refresh_token: &str) -> Result<TokenPair, AuthError>
```

Rotates a refresh token. The token is validated, blacklisted, and exchanged for a new pair. A token can only be rotated once.

## logout

```rust
pub async fn logout(&self, refresh_token: &str) -> Result<(), AuthError>
```

Blacklists a refresh token. Safe to call more than once.

## logout_all

```rust
pub async fn logout_all(&self, user_id: &str) -> Result<(), AuthError>
```

Voids every session issued so far for a user. See [Sessions and Revocation](sessions.md).

## request_password_reset

```rust
pub async fn request_password_reset(&self, email: &str) -> Result<(), AuthError>
```

Sends a reset email. Returns `Ok(())` for unknown emails so the endpoint cannot be used to discover accounts.

## reset_password

```rust
pub async fn reset_password(&self, token: &str, new_password: &str) -> Result<(), AuthError>
```

Sets a new password from a reset token and revokes every existing session.

## change_password

```rust
pub async fn change_password(
    &self,
    user_id: &str,
    old_password: &str,
    new_password: &str,
) -> Result<(), AuthError>
```

Changes the password for a logged in user and revokes every existing session.

## delete_account

```rust
pub async fn delete_account(&self, user_id: &str, password: &str) -> Result<(), AuthError>
```

Permanently deletes the account after re-checking the password.

## is_token_valid

```rust
pub async fn is_token_valid(&self, token: &str) -> Result<Claims, AuthError>
```

Validates an access token and returns its claims. Use it in your authentication middleware. Only access tokens pass. Refresh tokens are rejected.

## invalidate_user_cache

```rust
pub async fn invalidate_user_cache(&self, user_id: &str) -> Result<(), AuthError>
```

Clears the cached user entry. Call it when you change user data outside of authbox.

---

# Security features

* Argon2 password hashing on blocking threads
* Email verification before login
* Refresh token rotation with blacklisting
* Session revocation on password reset, password change, account deletion and logout all
* Atomic one-time tokens that cannot be used twice
* Separate token namespaces, so verification and reset tokens cannot be swapped
* Timing safe login for unknown emails
* Reset requests that never reveal whether an email exists
* Password length limits (12 to 128 characters)
* Emails trimmed and lowercased before lookup

---

# Next steps

1. [Creating a Registration DTO](registration-dto.md)
2. [Creating a User Type](user-model.md)
3. [Implementing a User Store](user-store.md)
4. [Building the Auth Service](auth-service.md)
5. [Actix Web Integration](actix-web.md)
6. [Production Deployment](production.md)
