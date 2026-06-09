# authbox overview

`authbox` is a lightweight, modular, async-first authentication framework for Rust.

It provides a complete authentication workflow while remaining fully framework-agnostic and storage-agnostic. Every major component is trait-driven, allowing applications to integrate their own databases, token systems, email providers, and storage backends.

At the center of the library is `AuthService`, which orchestrates authentication operations such as registration, login, token refresh, email verification, password recovery and is token valid.

---

# Core Philosophy

AuthBox follows three principles:

## Modular

Every authentication component is replaceable through traits.

You can use:

- PostgreSQL
- MySQL
- SQLite
- MongoDB
- Redis
- In-memory stores
- Custom backends

without changing authentication logic.

## Async First

All operations are designed around Rust's async ecosystem and work seamlessly with Tokio.

## Framework Agnostic

AuthBox does not depend on:

- Axum
- Actix
- Rocket
- Warp

This allows integration into any Rust application or service architecture.

---

# Building an AuthService

Authentication is configured using the builder API.

```rust
let auth = AuthService::builder()
    .store(user_store)
    .hasher(password_hasher)
    .tokens(token_manager)
    .blacklist(blacklist_store)
    .email_sender(email_provider)
    .email_templates(email_templates)
    .ott_store(one_time_token_store)
    .build();
```

The builder assembles all authentication dependencies into a single `AuthService` instance.

---

# AuthService Components

An `AuthService` consists of seven pluggable components:

| Component | Responsibility |
|------------|---------------|
| `UserStore` | User persistence and retrieval |
| `PasswordHasher` | Password hashing and verification |
| `TokenManager` | Access and refresh token generation |
| `TokenBlacklistStore` | Refresh token revocation |
| `EmailProvider` | Sending authentication emails |
| `EmailTemplateConfig` | Generating email content |
| `OneTimeTokenStore` | Temporary token storage |

```rust
pub struct AuthService<S, P, T, B, E, M, V> {
    pub store: S,
    pub hasher: P,
    pub tokens: T,
    pub blacklist: B,
    pub email_sender: E,
    pub email_templates: M,
    pub ott_store: V,
}
```

---

# Authentication Flow

A typical authentication lifecycle looks like:

```text
Register
   │
   ▼
Email Verification
   │
   ▼
Login
   │
   ▼
Access Token + Refresh Token
   │
   ▼
Refresh Token Rotation
   │
   ▼
Token Revocation
```

Password recovery is available at any point through the password reset flow.

---

# AuthService API

After creating an `AuthService`, the following authentication operations become available.

## register()

Creates a new user account.

```rust
pub async fn register(
    &mut self,
    input: S::RegisterDto,
) ->  Result<S::User, AuthError<S::Error, T::Error, B::Error>>

```

### What happens internally?

1. Check email if exists
2. Password is hashed.
3. User is persisted through `UserStore`.
4. A email verification token is generated.
5. The token is stored in `OneTimeTokenStore`.
6. A verification email is generated.
7. The email is sent through `EmailProvider`.
8. The created user is returned.

### Flow

```text
Register Request
        │
        ▼
Check if email already exists
        │--- email exists --> EmailAlreadyExists  Error
        ▼
Hash Password
        │
        ▼
Create User
        │
        ▼
Generate Verification Token
        │
        ▼
Store Token
        │
        ▼
Send Verification Email
        │
        ▼
Return User
```

---

## login()

Authenticates a user and issues tokens.

```rust
pub async fn login(
    &self,
    email: &str,
    password: &str,
) -> Result<T::Token, LoginError<T::Error, S::Error>>
```

### Validation Steps

1. Find user by email.
2. Verify email has been confirmed.
3. Verify password hash.
4. Generate authentication tokens.

### Possible Errors

```rust
pub enum LoginError<T, S> {
    InvalidCredentials,
    EmailNotVerified,
    Store(S),
    Token(T),
}
```

### Login Flow

```text
Find User
    │
    ▼
Email Verified?
    │
    ├── No → EmailNotVerified
    │
    ▼
Verify Password
    │
    ├── Invalid → InvalidCredentials
    │
    ▼
Generate Tokens
    │
    ▼
Return Tokens
```

---

## refresh_token()

Rotates a refresh token and issues a new token pair.

```rust
pub async fn refresh_token(
    &self,
    refresh_token: &str,
) -> Result<T::Token, AuthError<(), T::Error, B::Error>>
```

### What happens internally?

1. Verify refresh token.
2. Extract token claims.
3. Check blacklist.
4. Blacklist current refresh token.
5. Generate new token pair.


### Possible Errors


```rust

pub enum AuthError<S, T, B> {
    Store(S),
    Token(T),
    Blacklist(B),

    BlacklistedToken,
    EmailAlreadyExists,

    NotFound,
    InvalidToken,
    EmailAlreadyVerified,
}


```

### Refresh Flow

```text
Verify Refresh Token
          │
          ▼
Check Blacklist
          │
          ├── Blacklisted
          │         │
          │         ▼
          │   BlacklistedToken
          │
          ▼
Blacklist Current Token
          │
          ▼
Generate New Tokens
          │
          ▼
Return New Token Pair
```

---

## verify_email()

Marks a user's email address as verified.

```rust
pub async fn verify_email(&self, token: &str) -> Result<(), AuthError<S::Error, (), ()>>
```

### What happens internally?

1. Lookup token in `OneTimeTokenStore`.
2. Find associated user.
3. Mark email as verified.
4. Persist updated user.
5. Remove verification token.

### Flow

```text
Verification Token
         │
         ▼
Find User
         │
         ▼
Set Email Verified
         │
         ▼
Update User
         │
         ▼
Delete Token
```


## Possible Errors
```rust
AuthError::InvalidToken
AuthError::NotFound
AuthError::Store
AuthError::EmailAlreadyVerified
```


---

## request_password_reset()

Starts the password reset process.

```rust
pub async fn request_password_reset(
    &self,
    email: &str,
)
```

### What happens internally?

1. Find user by email.
2. Generate reset token.
3. Store token in `OneTimeTokenStore`.
4. Generate reset email.
5. Send email.

### Flow

```text
Find User
    │
    ▼
Generate Reset Token
    │
    ▼
Store Token
    │
    ▼
Send Reset Email
```

---

## reset_password()

Updates a user's password using a reset token.

```rust
  pub async fn reset_password(
        &self,
        token: &str,
        new_password: &str,
    ) -> Result<(), AuthError<S::Error, (), ()>>
)
```

### What happens internally?

1. Validate reset token.
2. Find associated user.
3. Hash new password.
4. Update stored password hash.
5. Delete reset token.

### Flow

```text
Validate Reset Token
          │
          ▼
Find User
          │
          ▼
Hash New Password
          │
          ▼
Update User
          │
          ▼
Delete Token
```


## Possible Errors
```rust
AuthError::InvalidToken
AuthError::NotFound
AuthError::Store
```

---

## is_token_valid()

 Checks if access token is valid

> 
>  `is_token_valid()` should be used in your authentication middlewares to verify access token

```rust
   pub async fn is_token_valid(
        &self,
        token: &str,
    ) -> Result<T::Claims, AuthError<(), T::Error, B::Error>>
```

### What happens internally?

1. Validate access  token.
2. Check token if blaclisted
3. Returns token claims

### Flow

```text
Validate Access Token
          │
          ▼
Check  token blacklisted
          │
          ▼
Returns token claims
```


## Possible Errors
```rust
AuthError::Token
AuthError::Blacklist
AuthError::BlacklistedToken
```

---

# Error Types

## LoginError

Returned by `login()`.

```rust
pub enum LoginError<T, S> {
    InvalidCredentials,
    EmailNotVerified,
    Store(S),
    Token(T),
}
```

| Variant | Description |
|----------|-------------|
| `InvalidCredentials` | Email or password is incorrect |
| `EmailNotVerified` | User has not verified their email |
| `Store(S)` | Storage layer failure |
| `Token(T)` | Token generation failure |

---

## AuthError

Returned by `refresh_token()`.

```rust
#[derive(Debug)]
pub enum AuthError<S, T, B> {
    Store(S),
    Token(T),
    Blacklist(B),

    BlacklistedToken,
    EmailAlreadyExists,

    NotFound,
    InvalidToken,
    EmailAlreadyVerified,
}
```

| Variant | Description |
|----------|-------------|
| `Token(T)` | Token verification or refresh failure |
| `Blacklist(B)` | Blacklist store failure |
| `BlacklistedToken` | Refresh token has already been revoked |
| `Store(S)` | Database store implementation erros |
| `EmailAlreadyExists` | Email already exists |
| `NotFound` | User not found |
| `InvalidToken` | OTT  for blacklists has already been revoked |
| `EmailAlreadyVerified` | User email already verified |
---

# Security Features

AuthBox includes several security mechanisms by default:

- Password hashing through `PasswordHasher`
- Email verification before login
- Refresh token rotation
- Refresh token revocation
- One-time token workflows
- Password reset protection
- Token blacklisting support
- Stateless JWT authentication

---

# Extensibility

Every authentication subsystem is trait-based.

You can provide custom implementations for:

- User storage
- Password hashing
- Token generation
- Email delivery
- Email templates
- Refresh token revocation
- One-time token storage

This allows AuthBox to adapt to virtually any application architecture while keeping authentication logic centralized in `AuthService`.

---

# Architecture Overview

```text
                ┌─────────────────┐
                │   AuthService   │
                └────────┬────────┘
                         │
      ┌──────────────────┼──────────────────┐
      │                  │                  │
      ▼                  ▼                  ▼

 UserStore      PasswordHasher      TokenManager
      │
      ▼

OneTimeTokenStore

      │
      ▼

EmailProvider
      +
EmailTemplateConfig

      │
      ▼

TokenBlacklistStore
```

---

# Next Steps

Continue with:

1. Creating a User Model
2. Implementing UserStore
3. Configuring JWT Authentication
4. Email Verification
5. Password Reset Flow
6. Refresh Token Rotation
7. Production Deployment
