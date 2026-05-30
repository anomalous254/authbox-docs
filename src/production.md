# Production Deployment

This guide covers best practices for deploying AuthBox in production environments.

---

## Recommended Setup

For production systems, use:

- PostgreSQL (UserStore)
- Redis (TokenBlacklistStore + OneTimeTokenStore)
- Argon2 (PasswordHasher)
- JWT with strong secret rotation (TokenManager)
- Real email provider (SendGrid / SES / Mailgun)


## Security Configuration

### Password Hashing

Use strong Argon2 settings:

- High memory cost
- Adequate iteration count
- Unique salt per password (handled automatically)


### JWT Tokens

- Use a long, random secret key
- Rotate secrets periodically
- Keep access tokens short-lived (15 min recommended)
- Use refresh token rotation + blacklist


### Token Blacklisting

Always enable blacklist storage in production:

- Prevents reuse of refresh tokens
- Protects against token theft
- Works best with Redis or distributed cache


### One-Time Tokens

Use TTL-based storage (Redis recommended):

- Email verification: 24 hours
- Password reset: 10–15 minutes


## Email Delivery

Use a real provider:

- AWS SES (recommended for scale)
- SendGrid
- Mailgun
- SMTP (small apps)

Ensure:

- SPF/DKIM configured
- Domain verified
- Rate limits handled



## Scaling Considerations

AuthBox is stateless by design:

- Multiple instances can run behind a load balancer
- Shared Redis required for:
  - blacklists
  - one-time tokens


## Deployment Checklist

- [ ] Secure JWT secret
- [ ] Enable refresh token rotation
- [ ] Use Redis for shared state
- [ ] Configure real email provider
- [ ] Use Argon2 hashing
- [ ] Enable HTTPS only
- [ ] Rate limit auth endpoints


## Summary

AuthBox is production-ready when combined with:

- Strong cryptography settings
- Shared state storage (Redis)
- Real email delivery system
- Proper secret management

It is designed to scale horizontally without changing core logic.
