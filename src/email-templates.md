# Configuring Email Templates

authbox separates email delivery from email content. `EmailProvider` sends the message. `EmailTemplateConfig` writes it.

This lets you change branding, wording or language without touching how mail is delivered.

---

## The trait

```rust
pub trait EmailTemplateConfig: Send + Sync {
    fn verify_email_subject(&self, user: &dyn AuthUser) -> String;
    fn verify_email_body(&self, user: &dyn AuthUser, token: &str) -> String;
    fn reset_password_subject(&self, user: &dyn AuthUser) -> String;
    fn reset_password_body(&self, user: &dyn AuthUser, token: &str) -> String;
}
```

The trait is not generic. Each method receives the user as `&dyn AuthUser`, plus the token for the body methods.

## Example

```rust
#[derive(Clone)]
pub struct MyTemplates;

impl EmailTemplateConfig for MyTemplates {
    fn verify_email_subject(&self, _user: &dyn AuthUser) -> String {
        "Verify your email".to_string()
    }

    fn verify_email_body(&self, user: &dyn AuthUser, token: &str) -> String {
        format!(
            "Hello {},

Please verify your email by clicking the link below:

http://localhost:8080/verify-email/{}

If you did not create this account, you can ignore this email.",
            user.email(),
            token
        )
    }

    fn reset_password_subject(&self, _user: &dyn AuthUser) -> String {
        "Reset your password".to_string()
    }

    fn reset_password_body(&self, user: &dyn AuthUser, token: &str) -> String {
        format!(
            "Hello {},

You requested a password reset.

Reset your password using the link below:

http://localhost:8080/reset-password/{}

If you did not request this, ignore this email.",
            user.email(),
            token
        )
    }
}
```

Replace the localhost links with your real frontend or API address.

## Things to keep in mind

* The verification link is valid for 24 hours and the reset link for 15 minutes. Say so in the email if you like.
* Tokens are secrets. Do not print them in production logs. The test fixtures print them to the console for convenience only.
* You can return HTML or plain text, depending on what your provider sends.
* You can personalize with the user's email or any data you can read from your own user type.

## Design

| Component | Responsibility |
|---|---|
| `EmailProvider` | Delivers the email |
| `EmailTemplateConfig` | Writes the subject and body |

Because they are separate, you can switch providers without touching templates, and support several languages with one provider.

## Summary

* Implement `EmailTemplateConfig` for verification and reset emails.
* Methods take `&dyn AuthUser` and the token.
* Templates are independent from the email provider.
