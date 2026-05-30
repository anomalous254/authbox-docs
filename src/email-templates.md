# Configuring Email Templates

AuthBox separates **email delivery** from **email content**.

The `EmailTemplateConfig` trait is responsible for generating email subjects and bodies for different authentication workflows.

---

## Responsibilities

Email templates are used for:

- Email verification content
- Password reset content

This allows full customization of email wording without changing the email provider logic.


## EmailTemplateConfig Trait

To define email templates, implement the following trait:

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


# Methods

## verify_email_subject

Generates the subject line for email verification.

```rust
fn verify_email_subject(&self, user: &U) -> String;
```

## verify_email_body

Generates the body content for email verification.

```rust
fn verify_email_body(
    &self,
    user: &U,
    token: &str,
) -> String;
```



## reset_password_subject

Generates the subject line for password reset emails.

```rust
fn reset_password_subject(&self, user: &U) -> String;
```


## reset_password_body

Generates the body content for password reset emails.

```rust
fn reset_password_body(
    &self,
    user: &U,
    token: &str,
) -> String;
```


# Example Implementation

```rust
#[derive(Clone)]
pub struct MockTemplates;

impl EmailTemplateConfig<User> for MockTemplates {
    fn verify_email_subject(&self, _: &User) -> String {
        "Verify Email".to_string()
    }

    fn verify_email_body(
        &self,
        _: &User,
        token: &str,
    ) -> String {
        format!("https:://yourapp/verif_email/token={}", token)
    }

    fn reset_password_subject(
        &self,
        _: &User,
    ) -> String {
        "Reset Password".to_string()
    }

    fn reset_password_body(
        &self,
        _: &User,
        token: &str,
    ) -> String {
        format!("https:://yourapp/reset_password/token={}", token)
    }
}
```


## Design Notes

- `EmailProvider` handles **sending emails**
- `EmailTemplateConfig` handles **email content**
- This separation allows:
  - Different email services
  - Different branding/templates per app
  - Easy localization (i18n support)
  - Clean separation of concerns

---

## Summary

- Implement `EmailTemplateConfig` to customize email content.
- Used for verification and password reset workflows.
- Works together with `EmailProvider` but does not depend on it.
- Fully flexible and backend-agnostic design.
