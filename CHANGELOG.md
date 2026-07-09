# Changelog

## 1.0.0

Initial release.

- **`Sqlite.Auth.Email`** — manages users' email addresses. Mirrors
  `Sqlite.Auth.Username`: `Raw`/`Normalized` phantom, `fromString`/`normalize`
  (trim + strict parse + lowercase)/`toString`, plus `new`, `change`
  (requires `Internal.Verified`), and `user` (lookup by email). The `emails`
  table enforces `user_id UNIQUE`, so a user holds at most one email.
- **`Sqlite.Auth.Email.Parser`** — parses RFC 5321-style ASCII email addresses
  for `Sqlite.Auth.Email.validate`, including dot-string and quoted-string local
  parts plus domain labels, IPv4 literals, and IPv6 literals.
- **`Sqlite.Auth.Otp`** — issues and verifies 8-character alphanumeric
  one-time passcodes for passwordless email login. `request` mints a code and
  returns the raw string exactly once (only its SHA-256 hash is stored at
  rest); `verify` consumes it on success and returns the `Internal.Verified`
  witness that `Sqlite.Auth.Session.issue` needs. Rate limiting in two layers:
  an issue throttle (`TooEarly { retryAt }`) and a verify lockout
  (`maxFailedAttempts` consecutive mismatches → `Locked`). `verify` takes a
  `Maybe Internal.User` — the value `Email.user` returns — so an unknown email
  and a wrong code both answer `Rejected` and both spend the same hashing
  work, matching `Sqlite.Auth.Password.validate`'s anti-enumeration stance.
  Expired unlocked rows are swept at the top of every `request`.
- New migrations `Sqlite.Auth.Email.migrationV1` and
  `Sqlite.Auth.Otp.migrationV1`. Both reference `users`, so run them after
  `Sqlite.Auth.User.migrationV1`.
