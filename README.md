# gren-sqlite-auth

`gren-sqlite-auth` is a simple way to add authentication primitives to your Gren application without getting caught with your pants down when handling credentials.

The ideal use for this package is to setup simple applications, prototypes, or small guarded areas of a website where authentication is useful and/or necessary, but doesn't need to be enterprise-grade. 

## What this package does

This package gives you username, email, and password authentication out-of-the-box without much work, including a full email verification flow — you only supply the sending of the mail. This is usually all you need for simple applications. Other credentials (one-time codes, etc.) may be added in the future.

More importantly, this package, when used properly (see _What this package does not do_ below) prevents you from mishandling credentials at the type-level. In other words: If you use functions in this package to handle credential information, it is difficult, if not impossible in some cases, to compromise the security of your application. This includes ensuring you handle credentials properly within your Gren program _and_ ensuring that any data saved to the database is done so as safely as possible.

## What this package does _not_ do

This package is not meant to provide production-grade authentication. While great care has gone into ensuring information is saved to the database properly, I (the author) am not a security engineer, nor has this codebase been reviewed by security professionals. As such, it's highly recommended to review and deeply interrogate the implementation of all functionality in this package before using it in any critical systems.

This package does not help in setting up proper endpoints for authentication or handling sessions with proper HTTP cookies and such. Modules to help with this - particularly in handling session tokens via HTTP cookies - may be added in the future.

Finally, this package does not prevent you from manipulating data saved to the database in a way that breaks the functionality provided by this package. In other words: Be very careful when writing your own SQL that interacts with the tables this package relies on. You have been warned!
