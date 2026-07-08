# gren-sqlite-auth

`gren-sqlite-auth` is a simple way to add authentication primitives to your Gren application without getting caught with your pants down when handling credentials.

The ideal use for this package is to setup simple applications, prototypes, or small guarded areas of a website where authentication is useful and/or necessary, but doesn't need to be enterprise-grade. 

## What this package does

This package gives you username and password authentication out-of-the-box without much work. This is usually all you need for simple applications. Other identifiers (email) and/or credentials (one-time codes, etc.) may be added in the future.

More importantly, this package, when used properly (see _What this package does not do_ below) prevents you from mishandling credentials at the type-level. In other words: If you use functions in this package to handle credential information, it is difficult, if not impossible in some cases, to compromise the security of your application. This includes ensuring you handle credentials properly within your Gren program _and_ ensuring that any data saved to the database is done so as safely as possible.

## What this package does _not_ do

This package is not meant to provide production-grade authentication. While great care has gone into ensuring information is saved to the database properly, I (the author) am not a security engineer, nor has this codebase been reviewed by security professionals. As such, it's highly recommended to review and deeply interrogate the implementation of all functionality in this package before using it in any critical systems.

This package issues and verifies session tokens (see `Sqlite.Auth.Session`), but it does not handle HTTP cookie transport. You must put the token into a cookie (or whatever transport you use) yourself. Helpers for cookie-based session transport may be added in the future.

Finally, this package does not prevent you from manipulating data saved to the database in a way that breaks the functionality provided by this package. In other words: Be very careful when writing your own SQL that interacts with the tables this package relies on. You have been warned!

## Setup

Install the package:

    gren package install joeybright/gren-sqlite-auth

Every entry point takes a `Sqlite.Database` (open one with `Sqlite.open`) and a `Crypto.SecureContext` (create one with `Crypto.getSecureContext`).

### Migrations

Run these in order — the last three reference the `users` table the first one creates:

    [ Sqlite.Auth.User.migrationV1
    , Sqlite.Auth.Username.migrationV1
    , Sqlite.Auth.Password.migrationV1
    , Sqlite.Auth.Session.migrationV1
    ]

Each migration is plug-and-play with `gren-sqlite-migrate`. If you use a different tool, read the SQL from each migration's `statements` field.

### Usage

Register a user with [`Sqlite.Auth.User.new`](Sqlite.Auth.User#new), attach a [`Sqlite.Auth.Username`](Sqlite.Auth.Username) and a [`Sqlite.Auth.Password`](Sqlite.Auth.Password), then issue a [`Sqlite.Auth.Session`](Sqlite.Auth.Session). See each module's docs for worked examples.
