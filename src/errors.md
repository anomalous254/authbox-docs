# Error Handling

Every `AuthService` method returns `Result<_, AuthError>`. Match on it in your handlers to pick the right HTTP status.

---

## AuthError

| Variant | Meaning |
|---|---|
| `InvalidCredentials` | Wrong email or password |
| `EmailNotVerified` | The password was correct but the email is not verified |
| `EmailAlreadyExists` | An account with that email already exists |
| `EmailAlreadyVerified` | The email was already verified |
| `InvalidToken` | The token is bad, expired, the wrong type, or revoked by a session cutoff |
| `BlacklistedToken` | The token was already revoked |
| `NotFound` | The user does not exist |
| Password length error | The password is not 12 to 128 characters (`AuthError::password_length()`) |
| `Store(e)` | Your `UserStore` failed |
| `Token(e)` | Token generation failed |
| `Blacklist(e)` | The blacklist store failed |
| `Ott(e)` | The one-time token store failed |
| `Email(e)` | The email provider failed |
| `Cache(e)` | The user cache failed during an invalidation |
| `Internal(e)` | An internal error, such as a failed blocking task |

## Which methods return what

| Method | Common errors |
|---|---|
| `register` | password length, `EmailAlreadyExists`, `Store`, `Ott`, `Email` |
| `verify_email` | `InvalidToken`, `NotFound`, `EmailAlreadyVerified`, `Store`, `Ott` |
| `login` | `InvalidCredentials`, `EmailNotVerified`, `Store`, `Token` |
| `me` | `InvalidToken`, `BlacklistedToken`, `NotFound`, `Store`, `Blacklist` |
| `refresh_token` | `InvalidToken`, `BlacklistedToken`, `Blacklist`, `Token` |
| `logout` | `InvalidToken`, `Blacklist` |
| `logout_all` | `Blacklist` |
| `request_password_reset` | `Store`, `Ott` |
| `reset_password` | password length, `InvalidToken`, `NotFound`, `Store`, `Blacklist` |
| `change_password` | password length, `InvalidCredentials`, `NotFound`, `Store`, `Blacklist` |
| `delete_account` | `InvalidCredentials`, `NotFound`, `Store`, `Blacklist` |
| `is_token_valid` | `InvalidToken`, `BlacklistedToken`, `Blacklist` |

## Suggested HTTP mapping

| Error | Status |
|---|---|
| `InvalidCredentials`, `InvalidToken`, `BlacklistedToken` | 401 Unauthorized |
| `EmailNotVerified` | 403 Forbidden |
| `EmailAlreadyExists`, `EmailAlreadyVerified` | 409 Conflict |
| Password length error | 400 or 422 |
| `NotFound` | 404 Not Found |
| `Store`, `Token`, `Blacklist`, `Ott`, `Email`, `Cache`, `Internal` | 500 Internal Server Error |

## Example

```rust
fn status_for(e: &AuthError) -> actix_web::http::StatusCode {
    use actix_web::http::StatusCode;

    match e {
        AuthError::InvalidCredentials
        | AuthError::InvalidToken
        | AuthError::BlacklistedToken => StatusCode::UNAUTHORIZED,
        AuthError::EmailNotVerified => StatusCode::FORBIDDEN,
        AuthError::EmailAlreadyExists
        | AuthError::EmailAlreadyVerified => StatusCode::CONFLICT,
        AuthError::NotFound => StatusCode::NOT_FOUND,
        _ => StatusCode::INTERNAL_SERVER_ERROR,
    }
}
```

## Tips

* Do not send `format!("{:?}", e)` to clients in production. Backend errors can contain internal details. Log them and return a short message.
* Use the same response for "unknown email" and "wrong password" on login. authbox already returns `InvalidCredentials` for both.
* `EmailNotVerified` is only returned after the password was correct, so it does not leak whether an email exists.
