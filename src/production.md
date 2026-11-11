# Production Deployment

This guide covers what to set up before running authbox in production.

---

## Recommended setup

| Component | Recommendation |
|---|---|
| Users | PostgreSQL through SQLx or Diesel |
| Blacklist and one-time tokens | Redis |
| Passwords | Argon2 (`DefaultHasher`) |
| Tokens | `DefaultJwtManager` with a strong secret |
| Email | A real provider (Resend, SendGrid, SES, Mailgun, or SMTP for small apps) |
| User cache | `DefaultMemoryUserCache`, or a shared cache for several instances |
| Runtime | Tokio |
| HTTP framework | Axum or Actix |

## Security configuration

### Database

* Put a UNIQUE index on the normalized email column.
* Make `create_user` report a unique violation as `AuthError::EmailAlreadyExists`.
* Store the email trimmed and lowercased.

### Passwords

* Use Argon2. It salts each password for you.
* Passwords must be 12 to 128 characters. authbox enforces this.

### JWT tokens

* Use a long random secret, at least 32 bytes.
* Load it from an environment variable or secret manager. Never commit it.
* Keep access tokens short lived (15 minutes is a good default).
* Keep `refresh_ttl_secs` in sync with your refresh token lifetime, if you write your own manager.
* Rotate the secret periodically. Changing it logs everyone out.

### Sessions

* Refresh token rotation and the blacklist are always on, so use Redis for shared state.
* Password reset, password change, account deletion and `logout_all` revoke every session automatically.
* Always take `user_id` from validated claims, never from request input.

### One-time tokens

* Email verification tokens last 24 hours.
* Password reset tokens last 15 minutes.
* Do not log tokens or put them in analytics.

### Cache

* Cache entries hold whatever `AuthUser::extra()` returns. Keep secrets out of it.
* With the in-memory cache, each instance has its own copy. Use a shared cache or a short TTL if stale data between instances is a problem.

## Email delivery

* Configure SPF and DKIM and verify your domain.
* Handle provider rate limits.
* Failed verification emails roll the registration back, so watch your email error logs.

## Observability

authbox logs through `tracing` and records cache counters through `metrics`:

* `authbox_cache_hits_total`
* `authbox_cache_misses_total`
* `authbox_cache_read_failures_total`
* `authbox_cache_write_failures_total`
* `authbox_cache_invalidate_failures_total`

Set up a `tracing` subscriber and a `metrics` exporter such as Prometheus.

## Scaling

`AuthService` keeps no state of its own, so you can run many instances behind a load balancer. All instances must share:

* the same database
* the same Redis (blacklist and one-time tokens)
* the same JWT secret

## Deployment checklist

- [ ] Strong JWT secret, loaded from the environment
- [ ] UNIQUE index on the email column
- [ ] Redis for the blacklist and one-time tokens
- [ ] A real email provider with SPF and DKIM
- [ ] HTTPS only
- [ ] Rate limiting on login, register and password reset endpoints
- [ ] Errors mapped to status codes, with no debug output sent to clients
- [ ] Tracing and metrics set up

## Summary

authbox is ready for production when it runs with strong secrets, shared Redis state, a real email service and a database with a unique email index. It scales horizontally without changes to your code.
