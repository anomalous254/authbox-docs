# Configuring Email Delivery

AuthBox sends emails through the `EmailProvider` trait.

The system is **provider-agnostic**, meaning you can:

- Use built-in providers for quick setup
- Implement your own custom provider
- Integrate with any email service

AuthBox does not depend on any specific email provider.

---

## Quick Start (Built-in Providers)

AuthBox includes ready-to-use providers for common email services.

### SendGrid

```rust
use authbox::prelude::SendGridEmailProvider;

let provider = SendGridEmailProvider::new(
    std::env::var("SENDGRID_API_KEY").unwrap(),
    std::env::var("SENDGRID_FROM_EMAIL").unwrap(),
    std::env::var("SENDGRID_FROM_NAME").unwrap(),
);
```

### Resend

```rust
use authbox::prelude::ResendEmailProvider;

let provider = ResendEmailProvider::new(
    std::env::var("RESEND_API_KEY").unwrap(),
    std::env::var("RESEND_FROM_EMAIL").unwrap(),
);
```

### SMTP

```rust
use authbox::prelude::SmtpEmailProvider;

let provider = SmtpEmailProvider::new(
    &std::env::var("SMTP_HOST").unwrap(),
    &std::env::var("SMTP_USERNAME").unwrap(),
    &std::env::var("SMTP_PASSWORD").unwrap(),
    &std::env::var("SMTP_FROM_EMAIL").unwrap(),
);
```

---

## The `EmailProvider` Trait

All email providers implement the same interface:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait EmailProvider {
    type Error;

    async fn send_email(
        &self,
        to: &str,
        subject: &str,
        body: &str,
    ) -> Result<(), Self::Error>;
}
```

This abstraction allows AuthBox to remain fully decoupled from any email service.

---

## Creating a Custom Email Provider

You can integrate any email service by implementing the `EmailProvider` trait.

### Example

```rust
use async_trait::async_trait;
use authbox::email::EmailProvider;

pub struct MyEmailProvider;

#[async_trait]
impl EmailProvider for MyEmailProvider {
    type Error = std::io::Error;

    async fn send_email(
        &self,
        to: &str,
        subject: &str,
        body: &str,
    ) -> Result<(), Self::Error> {
        println!("Sending email to {to}: {subject}");
        Ok(())
    }
}
```

---

## Mock Provider (Testing & Development)

For local development and testing, you can use a mock provider that prints emails to the console.

```rust
use async_trait::async_trait;
use authbox::email::EmailProvider;

#[derive(Clone)]
pub struct MockEmailProvider;

#[async_trait]
impl EmailProvider for MockEmailProvider {
    type Error = ();

    async fn send_email(
        &self,
        to: &str,
        subject: &str,
        body: &str,
    ) -> Result<(), Self::Error> {
        println!(
            "EMAIL TO: {}\nSUBJECT: {}\nBODY: {}",
            to, subject, body
        );

        Ok(())
    }
}
```

---

## Usage in AuthBox

Email providers are used throughout AuthBox for:

- Email verification
- Password reset emails
- Magic login links
- Account notifications

---

## Summary

- `EmailProvider` is a pluggable abstraction for email delivery.
- Built-in providers are available for SendGrid, Resend, and SMTP.
- You can integrate any email service by implementing the `EmailProvider` trait.
- Mock providers make local development and testing straightforward.
- AuthBox remains completely independent of any specific email service.
