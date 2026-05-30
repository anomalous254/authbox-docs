# Installation

Add the crate:

```bash
cargo add authbox
```

Import the prelude:

```rust
use authbox::prelude::*;
```

## Runtime

Authbox is async-first and requires Tokio.

```toml
tokio = { version = "1", features = ["full"] }
