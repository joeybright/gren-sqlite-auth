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

Workflows compose them:

- **`Auth.Simple.Register`** — create a user with an identifier and/or a stored
  credential, transactionally. Code-only users register just an identifier.
- **`Auth.Simple.Login`** — one `login` for every credential: resolve
  identifier → validate credential → issue token. Unknown identifiers and wrong
  secrets both cost a full validation round and both fail with the same
  `InvalidCredentials`, so callers can't be used to enumerate accounts.

The remaining verbs live on the concrete modules: `Code.request` mints a code
for you to deliver (the first phase of a code login, or of a reset), and
`Password.change` replaces a password after proving identity — the proof is
itself a credential: the current password (a change), or a one-time code (a
forgotten-password reset).

The package does no logging of its own — what (if anything) to record about
registrations, logins, and password changes is your application's decision.

## Tables

You own the `users` table — `Auth.Simple.migrationV1` bootstraps a minimal one
(`id`, `created_at`, `updated_at`) with `IF NOT EXISTS`, and you may extend it
freely. The package owns the rest, each foreign-keyed to `users.id`:
`usernames`, `passwords`, `codes`, and `sessions`. Run the migrations of the
pieces you use, base first:

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

-- Change a password by proving the current one...
changePassword { userId, current, newHashed } =
    Password.change database secureContext
        Password.credential
        { userId = userId, proof = Password.fromString current, new = newHashed }

-- ...or reset a forgotten one by proving with an emailed code.
resetPassword { userId, code, newHashed } =
    Password.change database secureContext
        Code.credential
        { userId = userId, proof = code, new = newHashed }

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
