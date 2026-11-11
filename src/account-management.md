# Password and Account Management

This chapter covers the methods for recovering and managing an account: password reset, password change and account deletion.

---

## Forgot password

Two steps. First the user asks for a reset email, then they use the token from it.

### Request a reset

```rust
auth.request_password_reset("user@example.com").await?;
```

* The email is trimmed and lowercased before lookup.
* A reset token is stored in the one-time token store and valid for 15 minutes.
* The reset email is sent using your `EmailTemplateConfig`.
* The method returns `Ok(())` even when the email is not registered, and even when the email fails to send (the failure is logged). The caller cannot tell whether an account exists, so always show the same message, such as "If that email is registered, a reset link is on its way".

### Reset with the token

```rust
auth.reset_password(&token, "a-new-long-password").await?;
```

1. The new password length is checked (12 to 128 characters).
2. The token is claimed with an atomic `take`, so it works once.
3. The password is hashed and saved.
4. Every existing session is revoked.
5. The user cache entry is cleared.

Possible errors include `InvalidToken` for an expired or used token, and `NotFound` if the user no longer exists.

## Change password

For a logged in user who knows the current password.

```rust
auth.change_password(&claims.sub, "old-password", "a-new-long-password").await?;
```

1. The new password length is checked.
2. The user is loaded and the old password is verified. A wrong one returns `InvalidCredentials`.
3. The new password is hashed and saved.
4. Every session is revoked, including the caller's own.
5. The user cache entry is cleared.

The client must log in again with the new password afterwards.

`user_id` must come from a validated access token, never from request input.

## Delete account

```rust
auth.delete_account(&claims.sub, "a-long-password").await?;
```

1. The user is loaded and the password is verified. A wrong one returns `InvalidCredentials`.
2. Every session is revoked.
3. The user is deleted from your store.
4. The user cache entry is cleared.

Sessions are revoked before the delete on purpose. If the delete fails, the user is just logged out and can try again. The other order could leave live tokens for an account that no longer exists.

`user_id` must come from a validated access token.

If your app has data linked to the user, such as posts or orders, clean it up in your own code. authbox only deletes the user through `UserStore::delete_user`.

## Email verification

```rust
auth.verify_email(&token).await?;
```

The token is claimed atomically. If the email is already verified, you get `EmailAlreadyVerified`. If a temporary failure happens after the claim, the token is restored so the user can click the link again.

## Summary

| Method | Needs login | Revokes sessions |
|---|---|---|
| `request_password_reset` | No | No |
| `reset_password` | No (uses a token) | Yes |
| `change_password` | Yes | Yes |
| `delete_account` | Yes | Yes |
| `verify_email` | No (uses a token) | No |
