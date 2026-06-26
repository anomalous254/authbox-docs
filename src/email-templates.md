# Configuring Email Templates

AuthBox separates **email delivery** from **email content**.

The `EmailProvider` is responsible for sending emails, while the `EmailTemplateConfig` trait generates the email subjects and bodies used throughout the authentication workflow.

This separation allows you to change your email branding and messaging without modifying how emails are delivered.

---

## Responsibilities

`EmailTemplateConfig` is used to generate:

- Email verification emails
- Password reset emails

Each template receives the authenticated user and any required token, allowing personalized messages and application-specific URLs.

---

## The `EmailTemplateConfig` Trait

Implement the following trait to customize email content:

```rust
pub trait EmailTemplateConfig<U: AuthUser> {
    fn verify_email_subject(&self, user: &U) -> String;

    fn verify_email_body(
        &self,
        user: &U,
        token: &str,
    ) -> String;

    fn reset_password_subject(&self, user: &U) -> String;

    fn reset_password_body(
        &self,
        user: &U,
        token: &str,
    ) -> String;
}
```

---

## Trait Methods

### `verify_email_subject`

Returns the subject line for email verification messages.

```rust
fn verify_email_subject(&self, user: &U) -> String;
```

### `verify_email_body`

Returns the body of the email verification message.

```rust
fn verify_email_body(
    &self,
    user: &U,
    token: &str,
) -> String;
```

---

### `reset_password_subject`

Returns the subject line for password reset emails.

```rust
fn reset_password_subject(&self, user: &U) -> String;
```

### `reset_password_body`

Returns the body of the password reset email.

```rust
fn reset_password_body(
    &self,
    user: &U,
    token: &str,
) -> String;
```

---

## Example Implementation

```rust
#[derive(Clone)]
pub struct MockTemplates;

impl EmailTemplateConfig<TestUser> for MockTemplates {
    fn verify_email_subject(&self, _: &TestUser) -> String {
        "Verify your email".to_string()
    }

    fn verify_email_body(
        &self,
        user: &TestUser,
        token: &str,
    ) -> String {
        format!(
            "Hello {},

Please verify your email by clicking the link below:

http://localhost:8080/verify-email/{}

If you did not create this account, you can ignore this email.",
            user.email,
            token
        )
    }

    fn reset_password_subject(&self, _: &TestUser) -> String {
        "Reset your password".to_string()
    }

    fn reset_password_body(
        &self,
        user: &TestUser,
        token: &str,
    ) -> String {
        format!(
            "Hello {},

You requested a password reset.

Reset your password using the link below:

http://localhost:8080/reset-password/{}

If you did not request this, ignore this email.",
            user.email,
            token
        )
    }
}
```

---

## Personalizing Emails

Because each method receives the authenticated user, templates can include personalized information such as:

- User name or email address
- Verification or reset links
- Company branding
- Support contact information
- Localized content
- HTML or plain text email bodies

For production applications, replace the example URLs with your application's frontend or API endpoints.

---

## Design

AuthBox intentionally separates responsibilities:

| Component | Responsibility |
|-----------|----------------|
| `EmailProvider` | Delivers emails using SMTP, SendGrid, Resend, or a custom provider |
| `EmailTemplateConfig` | Generates email subjects and message bodies |

This design provides several benefits:

- Clean separation of concerns
- Easy branding customization
- Localization (i18n) support
- HTML or plain-text templates
- Ability to switch email providers without changing templates

---

## Summary

- Implement `EmailTemplateConfig` to customize authentication emails.
- Templates are used for email verification and password reset workflows.
- The authenticated user and token are available when generating email content.
- Email templates are completely independent from the configured `EmailProvider`.
- Works with any supported email provider, including SMTP, SendGrid, Resend, or your own custom implementation.
