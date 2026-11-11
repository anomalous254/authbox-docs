# Installation

Add the crate:

```bash
cargo add authbox
```

Import the prelude:

```rust
use authbox::prelude::*;
```

The `authbox` crate re-exports the whole workspace through its prelude, so most projects only need this one dependency.

## Runtime

authbox is async-first and requires Tokio.

```toml
tokio = { version = "1", features = ["full"] }
```

## Services you will need

| Service | Used for | Required |
|---|---|---|
| A database | Your `UserStore` implementation | Yes |
| Redis | Token blacklist and one-time tokens (`authbox-redis`) | Yes, unless you write your own stores |
| An email service | Verification and reset emails (Resend, SendGrid, SMTP) | Yes |
| A cache | Faster `me()` lookups (`authbox-cache`) | No |

## Examples

A complete Actix Web project lives in the `examples` directory of the repository. See [Actix Web Integration](actix-web.md) for a walkthrough.
