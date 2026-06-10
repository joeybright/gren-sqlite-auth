# gren-sqlite-auth

- There are no utilities for helping you access or otherwise change any user
data records. It's assumed that any user-specific data is handled soley by the
consumer of this package and, ideally, is done in a separate table from the
users table handling auth. For example, if you want a user profile, create a
`user_profile` table with a foreign key relationship to the `users` table that
this package generates. This ensures you don't accidentaly edit or remove data
required to properly handle authentication.

Composable username/password authentication for Gren node applications, backed
by SQLite.

## How it's put together

Three small abstractions, each a record of functions, so new methods plug in
without touching the workflows:

- **`Auth.Simple.Identifier`** — something that names an account (a username
  today; an email or phone number tomorrow). `Auth.Simple.Identifier.Username`
  is the concrete one, normalizing (trim + lowercase) on both store and lookup.
- **`Auth.Simple.Credential`** — a secret the user proves they hold. The core
  is just `validate` + `burn` — what login needs — and every credential carries
  a kind-specific half beside it: **stored** credentials are durable secrets
  you write (`Auth.Simple.Credential.Password`: PBKDF2-SHA256, 600k iterations,
  PHC strings, lockout after repeated failures), **temporary** credentials are
  minted by the package and delivered by you (`Auth.Simple.Credential.Code`:
  6-digit single-use codes, 10-minute expiry, 5 guesses, one live code per
  user).
- **`Auth.Simple.Token`** — what a successful login issues.
  `Auth.Simple.Token.Session` stores sessions in SQLite, keeping only the
  SHA-256 hash of the token, and refuses expired or revoked sessions on verify.
  Every token species can also revoke everything a user holds at once — the
  hook password change and reset use to make sure a stolen session never
  survives them.

Workflows compose them:

- **`Auth.Simple.Register`** — create a user with an identifier and/or a stored
  credential, transactionally. Code-only users register just an identifier.
- **`Auth.Simple.Login`** — one `login` for every credential: resolve
  identifier → validate credential → issue token. Unknown identifiers and wrong
  secrets both cost a full validation round and both fail with the same
  `InvalidCredentials`, so callers can't be used to enumerate accounts.

The remaining verbs live on the concrete modules, and each temporary credential
carries a single intent: `Code.request` mints a code for you to deliver (the
first phase of a code login — codes only log in), `Password.change` replaces a
password after proving the current one, and `Password.requestReset` /
`Password.reset` handle the forgotten-password flow with their own dedicated
tokens — 32 random bytes, URL-safe, hashed at rest, 30-minute expiry,
single-use — meant to be emailed as a link. A reset token can't log in; a login
code can't reset.

The package does no logging of its own — what (if anything) to record about
registrations, logins, and password changes is your application's decision.

## Tables

You own the `users` table — `Auth.Simple.migrationV1` bootstraps a minimal one
(`id`, `created_at`, `updated_at`) with `IF NOT EXISTS`, and you may extend it
freely. The package owns the rest, each prefixed `auth_` and foreign-keyed to
`users.id`: `auth_usernames`, `auth_passwords`, `auth_password_reset_tokens`,
`auth_codes`, and `auth_sessions`. Run the migrations of the pieces you use,
base first:

```gren
migrations =
    [ Auth.Simple.migrationV1
    , Auth.Simple.Identifier.Username.migration.v1
    , Auth.Simple.Credential.Password.migrationV1
    , Auth.Simple.Credential.Code.migration.v1
    , Auth.Simple.Token.Session.migration.v1
    ]
```

## Example

```gren
import Auth.Simple.Credential.Code as Code
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
                            Password.stored
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

-- Email-code login, phase 1: mint a code for you to deliver.
-- Nothing means no such user — respond identically either way.
requestCode { username } =
    Code.request database secureContext
        Username.identifier
        { identifier = username }

-- Email-code login, phase 2: the same login function, different credential.
loginWithCode { username, code } =
    Login.login database secureContext
        Username.identifier
        Code.credential
        Session.token
        { identifier = username, secret = code }

-- Change a password by proving the current one. Every session the user
-- holds is revoked — including this device's — so log them in again after.
changePassword { userId, current, newHashed } =
    Password.change database secureContext
        Session.token
        { userId = userId, current = Password.fromString current, new = newHashed }

-- Forgot the password? Phase 1: mint a reset token to email as a link.
-- Nothing means no such user — respond identically either way.
requestPasswordReset { username } =
    Password.requestReset database secureContext
        Username.identifier
        { identifier = username }

-- Phase 2: redeem the token from the link. No userId needed — the token
-- resolves the user and returns { userId }. All the user's sessions are
-- revoked, so anything stolen dies with the reset.
resetPassword { token, newHashed } =
    Password.reset database secureContext
        Session.token
        { token = token, new = newHashed }

-- Verify a request: Nothing for unknown, expired, or revoked tokens.
-- A successful verify bumps the session's last_used_at.
authenticate cookie =
    Session.verifyToken { token = cookie } database secureContext

-- Logout: revoke the session (the row is kept, marked revoked).
logout cookie =
    Session.revoke { token = cookie } database secureContext
```

## Tests

```sh
cd tests && gren run Main
```
