# Architecture

AuthBox is designed as a **modular, trait-driven authentication framework** where every core responsibility is abstracted behind a pluggable interface.

---

## Core Design Principles

- **Separation of concerns** – authentication logic is separated from storage, email, and token handling
- **Dependency injection** – everything is composed via `AuthService::builder()`
- **Backend agnostic** – works with any database or external service
- **Testability** – all components can be mocked or replaced
- **Zero hard dependencies** – no required database, email provider, or token system


## High-Level Flow

A typical authentication request flows through these layers:

1. `AuthService` receives request
2. Calls `PasswordHasher` (login/register)
3. Uses `UserStore` for persistence
4. Uses `TokenManager` for JWT generation
5. Uses `EmailProvider` + `EmailTemplateConfig` for email flows
6. Uses `OneTimeTokenStore` for temporary tokens
7. Uses `TokenBlacklistStore` for refresh rotation & logout


## Component Architecture

AuthBox is built from interchangeable components:

- User layer → `UserStore`
- Security layer → `PasswordHasher`
- Session layer → `TokenManager`
- Revocation layer → `TokenBlacklistStore`
- Communication layer → `EmailProvider`
- Template layer → `EmailTemplateConfig`
- Temporary state layer → `OneTimeTokenStore`


## Key Insight

AuthBox does **not implement authentication directly**.

Instead, it orchestrates a set of independent components to build a complete authentication system.
