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

A login is a pipeline of three concrete steps, and the types make sure you
can't assemble them wrong:

1. **Resolve** an identifier — something that names an account (a username
   today; an email or phone number tomorrow).
   `Auth.Simple.Identifier.Username.resolve` maps a username (normalized: trim
   + lowercase on both store and lookup) to a `Maybe ResolvedUser`.
2. **Check** a credential — a secret the user proves they hold. Each kind
   carries its own `check`: `Auth.Simple.Credential.Password.check`
   (PBKDF2-SHA256, 600k iterations, PHC strings, lockout after repeated
   failures) for durable secrets you write, and
   `Auth.Simple.Credential.Code.check` (6-digit single-use codes, 10-minute
   expiry, 5 guesses, one live code per user) for secrets the package mints and
   you deliver. A check takes the *whole* `Maybe ResolvedUser` from resolve:
   handed `Nothing`, it burns the cost of a real validation and answers
   `Rejected`, so unknown identifiers cost the same as wrong secrets and
   callers can't be used to enumerate accounts.
3. **Issue** a token. `Auth.Simple.Token.Session.issue` stores a session in
   SQLite, keeping only the SHA-256 hash of the token, and refuses expired or
   revoked sessions on verify. Sessions can also be revoked all-at-once per
   user — the hook password change and reset use to make sure a stolen session
   never survives them.

The steps thread together through the witness types in **`Auth.Simple.Proof`**:
only an identifier module can mint a `ResolvedUser`, only a credential check
can mint the `Authenticated` proof inside its `Outcome`, and token issuance
demands that proof. A pipeline that skips a step — issuing a session for a user
that was merely *resolved*, never *authenticated* — does not compile.

Two workflows sit on top:

- **`Auth.Simple.Login`** — the three steps composed for you, with every
  failure collapsed into one uniform `Error` (`Rejected` from either cause
  lands as the same `InvalidCredentials`). Pipe the steps yourself when you
  need something it doesn't do.
- **`Auth.Simple.Register`** — create a user with an identifier and/or a stored
  credential, transactionally. Code-only users register just an identifier.

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

Every function takes its setup first — database, secure context, config — and
the value it acts on last, so you can partially apply your way to an
app-specific helper and then just pass tokens and usernames to it. Policy
(lockout, lifetimes, minimum lengths) lives in per-module `Config` records;
each module exports a `defaultConfig`, and a record update on it overrides one
knob:

```gren
import Auth.Simple.Credential.Code as Code
import Auth.Simple.Credential.Password as Password
import Auth.Simple.Identifier.Username as Username
import Auth.Simple.Login as Login
import Auth.Simple.Register as Register
import Auth.Simple.Token.Session as Session

-- Sessions here get a custom one-week lifetime instead of the 30-day default.
sessionConfig =
    { sessionLifetimeMillis = 7 * 24 * 60 * 60 * 1000 }

-- Register: validate and hash the new password, then write
-- user + username + password in one transaction.
register { username, password } =
    when Password.validateLength Password.defaultConfig (Password.fromString password) is
        Err tooShort ->
            Task.fail (ShowLengthError tooShort)

        Ok validated ->
            Password.hashWithDefaults secureContext validated
                |> Task.andThen
                    (\hashed ->
                        Register.register database secureContext
                            Username.identifier
                            Password.stored
                            { userId = Nothing, value = username, secret = hashed }
                    )

-- Login: returns { session, token } — hand `token` to the client
-- as a cookie, keep nothing else. Issuing also sweeps dead session rows.
login { username, password } =
    Login.login
        { resolve = Username.resolve database { username = username }
        , check =
            Password.check database secureContext Password.defaultConfig
                { password = Password.fromString password }
        , issue = Session.issue database secureContext sessionConfig
        }

-- Email-code login, phase 1: mint a code for you to deliver.
-- Nothing means no such user — respond identically either way.
requestCode { username } =
    Code.request database secureContext
        Code.defaultConfig
        Username.identifier
        { identifier = username }

-- Email-code login, phase 2: the same login composition, different check.
loginWithCode { username, code } =
    Login.login
        { resolve = Username.resolve database { username = username }
        , check = Code.check database secureContext Code.defaultConfig { code = code }
        , issue = Session.issue database secureContext sessionConfig
        }

-- Change a password by proving the current one. Every session the user
-- holds is revoked — including this device's — so log them in again after.
changePassword { userId, current, newHashed } =
    Password.change database secureContext
        Password.defaultConfig
        (Session.token sessionConfig)
        { userId = userId, current = Password.fromString current, new = newHashed }

-- Forgot the password? Phase 1: mint a reset token to email as a link.
-- Nothing means no such user — respond identically either way.
requestPasswordReset { username } =
    Password.requestReset database secureContext
        Password.defaultConfig
        Username.identifier
        { identifier = username }

-- Phase 2: redeem the token from the link. No userId needed — the token
-- resolves the user and returns { userId }. All the user's sessions are
-- revoked, so anything stolen dies with the reset.
resetPassword { token, newHashed } =
    Password.reset database secureContext
        (Session.token sessionConfig)
        { token = token, new = newHashed }

-- Verify a request: Nothing for unknown, expired, or revoked tokens.
-- A successful verify bumps the session's last_used_at. Partially applied,
-- so handlers just call `authenticate { token = cookie }`.
authenticate =
    Session.verifyToken database secureContext

-- Logout: revoke the session, then sweep dead rows out of the table.
logout cookie =
    Session.revoke database secureContext { token = cookie }
```

## Tests

```sh
cd tests && gren run Main
```
