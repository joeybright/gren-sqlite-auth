# gren-sqlite-auth

Add reasonably safe authentication to your Gren application.

    gren install joeybright/gren-sqlite-auth

## What this package does

This package gives you username and password authentication out-of-the-box for your Gren application. This is usually all you need for simple apps. Other identifiers and credentials (email, one-time codes, etc.) may be added in the future.

More importantly, this package handles authentication credentials safely: passwords, session and reset tokens are stored properly hashed. Opaque types make it hard to mishandle a credential in your Gren code. For example, it's not possible, using functions in this package, to save a plaintext password to the database.

This package lets you extend the application tables as you deem fit for your use case. Most functions require a minimal base amount of tables and schemas to operate properly, of which you can build on if required.

## What this package does _not_ do

While great care has gone into ensuring information is saved to the database properly, I (the author) am not a security engineer, nor has this codebase been reviewed by security professionals. As such, it's highly recommended to review and deeply interrogate the implementation of all functionality in this package before using it in any critical systems.

This package does not wire up your authentication endpoints, build the HTTP server for you, or help construct proper HTTP responses. Additional functionality of this sort may be added in the future.

Finally, this package does not prevent you from manipulating data saved to the database in a way that breaks the functionality provided by this package. In other words: Be very careful when writing your own SQL that interacts with the tables this package relies on.

## Handling credentials

This package currently does not help in constructing HTTP responses and storing credentials properly for a user in their browser. It's of the utmost importance to properly handle session and reset tokens when sending them over the network. Some guidance for doing so is below:

### Sessions and cookies

It's highly recommended you store the raw token from [`Session.issue`](Sqlite.Auth.Session#issue) as a cookie:

    Set-Cookie: __Host-session={session-token}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000

- `__Host-` stops subdomains shadowing the cookie.
- `Path=/` is required by `__Host-` and scopes the cookie to the whole site.
- `HttpOnly` stops XSS from reading the token with JavaScript.
- `Secure` sends the cookie only over HTTPS, so the network can't read it.
- `SameSite=Lax` stops cross-site request forgery (CSRF); keep GETs side-effect-free.
- `Max-Age=2592000` should match the lifetime of the issued session token. However, session expiration is still enforced server-side.

A cookie with these properties prevents malicious actors from gaining access to the session token and impersonating a logged-in user.

Whenever you log out a user, send a new session cookie down to their client with `Max-Age=0` to ensure their client no longer has an active session cookie.
