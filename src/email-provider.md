# Configuring Email Delivery

AuthBox sends emails using the `EmailProvider` trait.  

The framework is **provider-agnostic** and does not depend on a specific email service.

---

## Supported Providers

Common providers you can integrate with:

- SendGrid
- Resend
- Mailgun
- AWS SES
- SMTP
- Custom providers


## EmailProvider Trait

To send emails, implement the following trait:

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

## Used By AuthBox

AuthBox sends emails for workflows such as:

- Email verification
- Password reset
- One-time authentication links (magic links)


## Example: Mock Email Sender

For development or testing, you can implement a simple mock provider:

```rust
use async_trait::async_trait;

#[derive(Clone)]
pub struct MockEmailSender;

#[async_trait]
impl EmailProvider for MockEmailSender {
    type Error = ();

    async fn send_email(
           &self,
        to: &str,
        subject: &str,
        body: &str,
    ) -> Result<(), Self::Error> {
        println!(
            "\nEMAIL TO: {}\nSUBJECT: {}\nBODY: {}",
            to, subject, body
        );

        Ok(())
    }
}
```

- This prints emails to the console instead of sending them.  
- Useful for local development and automated tests.  


## Production Email Providers

For production, implement the trait using real email services like:

- SendGrid
- Mailgun
- AWS SES
- SMTP servers

This allows AuthBox to send actual email messages without changing the core logic.

---

## Summary

- `EmailProvider` is a pluggable interface for sending emails.  
- You can implement a mock provider for testing or integrate a real provider for production.  
- AuthBox uses email for verification, password reset, and other authentication flows.
