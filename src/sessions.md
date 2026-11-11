# Sessions and Revocation

authbox can end sessions in three ways: one token, one device, or every device.

---

## Logging out of one device

```rust
auth.logout(&refresh_token).await?;
```

The refresh token is blacklisted. Calling it again with the same token is fine, so a client that retries will not get an error.

Access tokens are short lived and stay valid until they expire. Keep their lifetime short (15 minutes is a good default).

## Logging out of every device

```rust
auth.logout_all(&claims.sub).await?;
```

This voids every session issued so far for that user. Tokens issued after the call, for example from a fresh login, work normally.

`user_id` must come from a validated access token (`claims.sub`), never from request input.

## How it works

authbox stores a cutoff time for the user in the blacklist store, using the current time in unix seconds. Every call to `is_token_valid` and `refresh_token` compares the token's `iat` (issued at) with that cutoff. If `iat` is at or before the cutoff, the token is rejected with `InvalidToken`.

This means:

* One write voids all of a user's tokens, however many devices there are.
* There is no need to list or track individual tokens.
* The note is kept for the refresh token lifetime (`TokenManager::refresh_ttl_secs`). After that every older token has expired anyway, so it is dropped.
* A token issued in the same second as the revocation counts as revoked, because the comparison is "at or before".

## Automatic revocation

These actions revoke every session on their own:

| Action | Why |
|---|---|
| `reset_password` | The old password may have been compromised |
| `change_password` | Other devices should log in again with the new password |
| `delete_account` | No live tokens should remain for a deleted account |

After `reset_password` or `change_password`, the client has to log in again.

## Refresh race protection

`refresh_token` checks the cutoff twice. Once before rotating, and again after the new pair is generated. If the user was revoked in that small window, the new pair is discarded and `InvalidToken` is returned.

## Using it in middleware

Calling `is_token_valid` in your middleware is enough to get all of this. It checks the signature, expiry, token type, blacklist and session cutoff in one call. See [Actix Web Integration](actix-web.md).

## Summary

* `logout` ends one refresh token.
* `logout_all` ends every session.
* Password reset, password change and account deletion revoke all sessions automatically.
* Revocation needs a `TokenBlacklistStore`, so use Redis in production.
