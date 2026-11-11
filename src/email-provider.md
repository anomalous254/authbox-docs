# Configuring Email Delivery

authbox sends email through the `EmailProvider` trait. It is provider agnostic: use a built-in provider, or implement the trait for any service.

---

## Built-in providers

All three are available from `authbox::prelude`.

### Resend

```rust
let provider = ResendEmailProvider::new(
    std::env::var("RESEND_API_KEY").unwrap(),
    std::env::var("RESEND_FROM_EMAIL").unwrap(),
);
```

### SendGrid

```rust
let provider = SendGridEmailProvider::new(
    std::env::var("SENDGRID_API_KEY").unwrap(),
    std::env::var("SENDGRID_FROM_EMAIL").unwrap(),
    std::env::var("SENDGRID_FROM_NAME").unwrap(),
);
```

### SMTP

```rust
let provider = SmtpEmailProvider::new(
    &std::env::var("SMTP_HOST").unwrap(),
    &std::env::var("SMTP_USERNAME").unwrap(),
    &std::env::var("SMTP_PASSWORD").unwrap(),
    &std::env::var("SMTP_FROM_EMAIL").unwrap(),
);
```

Example environment variables:

```env
SMTP_HOST=smtp.gmail.com
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM_EMAIL=AuthBox <your-email@gmail.com>
```

> For Gmail, use an App Password and not your account password.

Pass the provider to the builder:

```rust
.email_sender(provider)
```

## The trait

```rust
#[async_trait]
pub trait EmailProvider: Send + Sync {
    async fn send_email(&self, to: &str, subject: &str, body: &str) -> Result<(), BoxError>;
}
```

## Custom provider

```rust
pub struct MyEmailProvider;

#[async_trait]
impl EmailProvider for MyEmailProvider {
    async fn send_email(&self, to: &str, subject: &str, body: &str) -> Result<(), BoxError> {
        println!("Sending email to {to}: {subject}");
        Ok(())
    }
}
```

## Mock provider for development

Print emails to the console instead of sending them:

```rust
#[derive(Clone)]
pub struct MockEmailProvider;

#[async_trait]
impl EmailProvider for MockEmailProvider {
    async fn send_email(&self, to: &str, subject: &str, body: &str) -> Result<(), BoxError> {
        println!("EMAIL TO: {to}\nSUBJECT: {subject}\nBODY: {body}");
        Ok(())
    }
}
```

## What happens when sending fails

* During `register`, a failed send rolls back the new user and the verification token, and `register` returns `AuthError::Email`.
* During `request_password_reset`, a failed send is logged and not returned. The caller always gets `Ok(())`, so the endpoint cannot be used to find out which emails are registered.

## Summary

* Resend, SendGrid and SMTP are built in.
* Implement `EmailProvider` for anything else.
* Providers are interchangeable without changing your templates.
