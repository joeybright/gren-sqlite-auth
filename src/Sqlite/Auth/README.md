...

Username.getUserId db secureContext "username" -- Get the username
    |> Password.validate (Password.fromString "passed password")  -- existing user; phantom type?
    |> Session.issue 

Email.getUserId db secureContext "email@domain.com"
    |> Code.issue db secureContext
    |> ... send email, other tasks

User.checkId db secureContext "uuid" -- returns `User Exists`
    |> Username.add db secureContext "username" -- returns user id?
    |> Password.add

identifier - `Task` produces a

---

<!-- Create a user, add a username and password, and issue a session -->
User.new db secureContext -- `Task x (User Exists)`
    <!-- -- Sqlite.Database -> Crypto.SecureContext -> String -> User Exists -> Task x (User Exists) -->
    |> Task.andThen (Username.new db secureContext "new_username")
    <!-- -- Sqlite.Database -> Crypto.SecureContext -> Password_ Hashed -> User Exists -> Task x (User Exists) -->
    |> Task.andThen (Password.add db secureContext hashedPassword)
    <!-- -- Sqlite.Database -> Crypto.SecureContext -> User Exists -> Task x Session.Token -->
    |> Task.andThen (Session.issue db secureContext)

User.new db secureContext
    <!-- -- Sqlite.Database -> Crypto.SecureContext -> Task x Code.Token -->
    |> Task.andThen (Code.issue db secureContext)
    |> Task.andThen someSendEmailApplicationCode

Username.getUserId db secureContext "username" -- `Task x (User Exists)`

---

- Create account / identity / credential
- Login with identity and credential (issue session)
- Reset credential
- ...
