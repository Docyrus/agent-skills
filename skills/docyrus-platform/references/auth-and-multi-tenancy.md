# Authentication & Multi-Tenancy

## Authentication Methods

- **OAuth2 Authorization Code (PKCE)** — Browser-based apps
- **OAuth2 Client Credentials** — Server-to-server
- **OAuth2 Device Code** — CLI and headless environments
- **Automatic token refresh** — Transparent renewal with refresh tokens
- **React auth provider (`@docyrus/signin`)** — Supports standalone PKCE and iframe `postMessage` auth modes, auto-fetches the current user from `/v1/users/me`, and can refresh user state after role or permission changes

## Access Control

- **Scope-based permissions** — ReadWrite.All, User.Read, Architect.ReadWrite.All, AI.ReadWrite.All, MCP.Connect, and more
- **Tenant isolation** — Enforced data separation between tenants on every query
- **Role-based access** — Roles with granular permission sets; React apps can check them with `hasRole()` from `useDocyrusAuth()` or `@docyrus/signin/core`
- **Permission helpers** — React apps can evaluate ACL permission checks with `hasPermission(operation, dataSourceId)` using the merged `aclRules` returned for the current user
- **Record-level ACL** — Per-record ownership and sharing rules (private, read-only, read/update, full access)
- **Field-level ACL** — Restrict visibility or editability of individual fields per role

## React App Authorization Helpers

- `useDocyrusAuth()` exposes `user`, `hasRole`, `hasPermission`, and `refreshUser`
- `user` is auto-fetched from `/v1/users/me` after authentication completes
- Pure framework-agnostic helpers are also available from `@docyrus/signin/core`

## User Management

- User signup with email verification, profile management, avatar, preferences
- Password change/reset, email change workflows
- Multi-device support with device registration for push notifications
- Per-user AI feature access control

## Multi-Tenancy

- Fully isolated data per tenant
- Multiple apps per tenant, each with scoped data sources
- Tenant-specific configurations, branding, and translations
- Organizational hierarchy with teams, roles, and user scoping

## Portal System

- Client portal configuration with dedicated portal users, organizations, roles, and sessions
- Separate authentication and access control for external users
