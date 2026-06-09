# gren-sqlite-auth

Composable username/password authentication for Gren node applications, backed
by SQLite.

## How it's put together

Three small abstractions, each a record of functions, so new methods plug in
without touching the workflows:

- **`Auth.Simple.Identifier`** — something that names an account (a username
  today; an email or phone number tomorrow). `Auth.Simple.Identifier.Username`
  is the concrete one, normalizing (trim + lowercase) on both store and lookup.
- **`Auth.Simple.Credential`** — a secret the user proves they hold (a password
  today; a one-time code tomorrow). `Auth.Simple.Credential.Password` hashes
  with PBKDF2-SHA256 (600k iterations, PHC strings) and enforces lockout after
  repeated failures.
- **`Auth.Simple.Token`** — what a successful login issues.
  `Auth.Simple.Token.Session` stores sessions in SQLite, keeping only the
  SHA-256 hash of the token, and refuses expired or revoked sessions on verify.

Workflows compose them:

- **`Auth.Simple.Register`** — create a user with an identifier and/or
  credential, transactionally.
- **`Auth.Simple.Login`** — resolve identifier → validate credential → issue
  token. Unknown identifiers and wrong secrets both cost a full hashing round
  and both fail with the same `InvalidCredentials`, so callers can't be used to
  enumerate accounts.
- **`Auth.Simple.Change`** — replace a credential after proving the current one.
- **`Auth.Simple.Credential.Password.Reset`** — single-use, expiring reset
  tokens for the forgotten-password case (delivery is your job).

The package does no logging of its own — what (if anything) to record about
registrations, logins, and password changes is your application's decision.

## Tables

You own the `users` table — `Auth.Simple.migrationV1` bootstraps a minimal one
(`id`, `created_at`, `updated_at`) with `IF NOT EXISTS`, and you may extend it
freely. The package owns the rest, each foreign-keyed to `users.id`:
`usernames`, `passwords` + `password_reset_tokens`, and `sessions`. Run the
migrations of the pieces you use, base first:

```gren
migrations =
    [ Auth.Simple.migrationV1
    , Auth.Simple.Identifier.Username.migration.v1
    , Auth.Simple.Credential.Password.migrationV1
    , Auth.Simple.Token.Session.migration.v1
    ]
```

## Example

```gren
import Auth.Simple.Credential.Password as Password
import Auth.Simple.Identifier.Username as Username
import Auth.Simple.Login as Login
import Auth.Simple.Register as Register
import Auth.Simple.Token.Session as Session

-- Register: validate and hash the new password, then write
-- user + username + password in one transaction.
register { username, password } =
    when Password.validateLength (Password.fromString password) is
        Err tooShort ->
            Task.fail (ShowLengthError tooShort)

        Ok validated ->
            Password.hashWithDefaults validated secureContext
                |> Task.andThen
                    (\hashed ->
                        Register.register database secureContext
                            Username.identifier
                            Password.credential
                            { userId = Nothing, value = username, secret = hashed }
                    )

-- Login: returns { session, token } — hand `token` to the client
-- as a cookie, keep nothing else.
login { username, password } =
    Login.login database secureContext
        Username.identifier
        Password.credential
        Session.token
        { identifier = username, secret = Password.fromString password }

-- Verify a request: Nothing for unknown, expired, or revoked tokens.
-- A successful verify bumps the session's last_used_at.
authenticate cookie =
    Session.verifyToken cookie database secureContext

-- Logout: revoke the session (the row is kept, marked revoked).
logout cookie =
    Session.revoke cookie database secureContext
```

## Tests

```sh
cd tests && gren run Main
```
