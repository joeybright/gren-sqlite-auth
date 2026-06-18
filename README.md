# gren-sqlite-auth

`gren-sqlite-auth` is a simple way to add authentication primitives to your Gren application without getting caught with your pants down when handling credentials.

The ideal use for this package is to setup simple applications, prototypes, or small guarded areas of a website where authentication is useful and/or necessary, but doesn't need to be enterprise-grade. 

## What this package does

This package gives you username and password authentication out-of-the-box without much work. This is usually all you need for simple applications. Other identifiers (email) and/or credentials (one-time codes, etc.) may be added in the future.

More importantly, this package, when used properly (see _What this package does not do_ below) prevents you from mishandling credentials at the type-level. In other words: If you use functions in this package to handle credential information, it is difficult, if not impossible in some cases, to comprimise the security of your application. This includes ensuring you handle credentials properly within your Gren program _and_ ensuring that any data saved to the database is done so as safely as possible.

## What this package does _not_ do

This package is not meant to provide production-grade authencation. While great care has gone into ensuring information is saved to the database properly, I (the author) am not a security engineer, nor has this codebase been reviewed by security professionals. As such, it's highly recommended to review and deeply interrogate the implemention of all functionality in this package before using it in any critical systems.

This package does not help in setting up proper endpoints for authentication or handling sessions with proper HTTP cookies and such. Modules to help with this - particularly in handling session tokens via HTTP cookies - may be added in the future.

Finally, this package does not prevent you from manipulating data saved to the database in a way that breaks the functionality provided by this package. In other words: Be very careful when writing your own SQL that interacts with the tables this package relies on. You have been warned!

## Getting started

This package gives you small, composable pieces — create a user, attach a
username, store a password, validate a login, issue and verify sessions — that
you wire into your own flows. Two things hold throughout:

- **Tables come from migrations.** Each module exposes a `migrationV1`; run them
  in order — `User`, then `Username`, `Password`, and `Session` (the later
  tables reference `users`).
- **Sensitive steps demand proof.** Changing a password or username, or issuing
  a session, requires a `Verified` value. Only a password check
  (`Password.validate`) or a live session (`Session.verify`) can mint one, so you
  can never act on a user who has merely been looked up.

The examples below assume these imports:

```gren
import Crypto
import FileSystem
import FileSystem.Path
import Sqlite
import Sqlite.Migrate
import Sqlite.Auth.User as User
import Sqlite.Auth.Username as Username
import Sqlite.Auth.Password as Password
import Sqlite.Auth.Session as Session
import Task exposing ( Task )
```

### Opening the database and running migrations

You need a `FileSystem.Permission` and a `Crypto.SecureContext` (obtained once
at startup with `FileSystem.initialize` and `Crypto.getSecureContext`). With
those, open the database and migrate it. Migrations here use the
[`gren-sqlite-migrate`](https://packages.gren-lang.org) package; each
`migrationV1` is a plain `{ name, statements }` record, so you can also run the
SQL yourself if you prefer.

```gren
setUp : FileSystem.Permission -> Crypto.SecureContext -> Task AppError Sqlite.Database
setUp fsPermission secureContext =
    Sqlite.open fsPermission Sqlite.defaultOptions (FileSystem.Path.fromPosixString "app.sqlite")
        |> Task.mapError DatabaseError
        |> Task.andThen
            (\database ->
                Sqlite.Migrate.migrate database
                    { secureContext = secureContext }
                    [ User.migrationV1
                    , Username.migrationV1
                    , Password.migrationV1
                    , Session.migrationV1
                    ]
                    |> Task.mapError MigrateError
            )
```

### One error type for the app

Each step fails with its own error type, so it helps to collect them into one
type for your own code and map into it as you go:

```gren
type AppError
    = DatabaseError Sqlite.Error
    | MigrateError Sqlite.Migrate.Error
    | HashError Password.HashError
    | ValidateError Password.ValidateError
    | VerifyError Session.VerifyError
    | InvalidUsername
    | CredentialsRejected
```

### Registering a user

Normalize the username, hash the password, then create the user, attach the
username, and store the password in a single transaction. Here we log the new
user straight in by issuing a session:

```gren
register :
    Sqlite.Database
    -> Crypto.SecureContext
    -> { username : String, password : String }
    -> Task AppError { session : Session.Session, token : String }
register database secureContext { username, password } =
    when Username.normalize (Username.fromString username) is
        Nothing ->
            Task.fail InvalidUsername

        Just name ->
            Password.hash secureContext Password.defaultOptions (Password.fromString password)
                |> Task.mapError HashError
                |> Task.andThen
                    (\hashed ->
                        Sqlite.inTransaction database
                            (User.new database secureContext
                                |> Task.andThen (Username.new database { username = name })
                                |> Task.andThen (Password.add database hashed)
                            )
                            |> Task.mapError DatabaseError
                    )
                |> Task.andThen
                    (\_ -> login database secureContext { username = username, password = password })
```

### Logging in

Resolve the username to a user, check the password, and — only on a verified
match — issue a session. Hand `token` to the client exactly once; the database
stores only its hash.

```gren
login :
    Sqlite.Database
    -> Crypto.SecureContext
    -> { username : String, password : String }
    -> Task AppError { session : Session.Session, token : String }
login database secureContext { username, password } =
    when Username.normalize (Username.fromString username) is
        Nothing ->
            Task.fail InvalidUsername

        Just name ->
            Username.user database { username = name }
                |> Task.mapError DatabaseError
                |> Task.andThen
                    (\maybeUser ->
                        Password.validate database secureContext Password.defaultOptions
                            (Password.fromString password)
                            maybeUser
                            |> Task.mapError ValidateError
                    )
                |> Task.andThen
                    (\outcome ->
                        when outcome is
                            Password.Verified proof ->
                                Session.issue database secureContext Session.defaultOptions proof
                                    |> Task.mapError DatabaseError

                            Password.Rejected ->
                                Task.fail CredentialsRejected

                            Password.Locked ->
                                Task.fail CredentialsRejected
                    )
```

### Authorizing a request with a live session

On each request, verify the session token. `Session.verify` succeeds with the
session and a `verified` proof, which you can pass straight to a step that
demands it — for example, renaming the user:

```gren
rename :
    Sqlite.Database
    -> Crypto.SecureContext
    -> { token : String, newUsername : String }
    -> Task AppError {}
rename database secureContext { token, newUsername } =
    when Username.normalize (Username.fromString newUsername) is
        Nothing ->
            Task.fail InvalidUsername

        Just name ->
            Session.verify database secureContext Session.defaultOptions { token = token }
                |> Task.mapError VerifyError
                |> Task.andThen
                    (\{ verified } ->
                        Username.change database { newUsername = name } verified
                            |> Task.mapError DatabaseError
                    )
                |> Task.map (\_ -> {})
```

If the token names no live session, `verify` fails with the matching
`Session.VerifyError` — `TokenNotFound`, `Revoked`, `Expired`, or `IdleTimeout`.

### Logging out

Revoke the session behind a token:

```gren
logout :
    Sqlite.Database
    -> Crypto.SecureContext
    -> { token : String }
    -> Task AppError {}
logout database secureContext { token } =
    Session.revoke database secureContext Session.defaultOptions { token = token }
        |> Task.mapError DatabaseError
```

### Where to go next

- **Change a password** — `Password.change`, gated by a `Verified` proof (revokes
  the user's other sessions).
- **Forgotten password** — `Password.requestReset` mints a single-use token to
  email out; `Password.reset` redeems it, sets the new password, revokes existing
  sessions, and returns a `Verified` proof so you can log the user back in.
- **Revoke everywhere** — `Session.revokeAll` ends every session a user holds.
- Tune behavior through the `Options` records: `Password.defaultOptions`
  (PBKDF2 iterations, lockout) and `Session.defaultOptions` (lifetime, idle
  timeout). See each module's documentation for the full API.
