# Changelog

## 1.0.0

### Added

- Initial release of `Sqlite.Auth`: username and password authentication for
  SQLite-backed Gren apps.
- `Sqlite.Auth.User` for creating, looking up, and deleting users.
- `Sqlite.Auth.Username` for creating, normalizing, looking up, renaming, and
  deleting usernames.
- `Sqlite.Auth.Password` for hashing, storing, validating, changing, and
  deleting passwords, plus a request/reset token flow and failed-login
  lockouts.
- `Sqlite.Auth.Session` for issuing, verifying, and revoking login sessions,
  with expiry and idle timeouts.
- Properly hashed password storage with PBKDF2 passwords and SHA-256 session and reset
  tokens, behind opaque types that prevent plain-text storage.
- Plug-and-play `migrationV1` migrations for `gren-sqlite-migrate`.
