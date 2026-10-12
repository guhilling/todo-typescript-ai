# Authentication: OIDC with Okta, Keycloak and Cognito

Status: accepted as direction (Gunnar, 2026-10-13); details open until the domain model exists.

## Decision

- **The backend speaks plain OpenID Connect (OIDC)** and is not tied to one identity provider.
  Provider differences are configuration (issuer URL and client settings per environment), not
  code.
- **Okta** is the identity provider the project will use.
- **Keycloak locally** (container in the development setup).
- **AWS Cognito in qa-testing.**

## Consequences

- Token validation uses the issuer's discovery document and JWKS, so the same code runs against
  all three providers. Claims the code depends on (subject, roles or groups) must exist in all
  three; where they differ, a mapping lives in one place.
- Role and group claims are the likely difference between Okta, Keycloak and Cognito. This is not
  yet checked and has to be settled before authorisation rules are written.

## Open

- Which environments use Okta (beyond local and qa-testing) is not stated yet.
- Library choice (for example `jose` for token validation, or a Fastify plugin) is decided when
  the first protected endpoint is written, and checked against current documentation first (see
  [ai-generated-code.md](ai-generated-code.md)).
- Authorisation model (roles, ownership of todos): decided with the domain model.
