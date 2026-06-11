Doing in-depth code review, file-by-file, line-by-line.

This includes review code _and_ comments, especially comments that are public in the published package.

`src/Auth.Simple`

- [x] Remove the pragma check (including `src/Utils.Sqlite.PragmaTableXInfo.gren` file)
- [x] Write out proper documentation given this is the first exposure people have to the entire package and the requirement for using it

`src/Auth.Simple.User`

- [x] Better documentation

`src/Auth.Simple.Token`

- [x] Better documentation
- [ ] `revokeForAll` -> `revokeAll`
- [ ] Review the revoke logic and other logic, given there is no associated user id? Is that not... bad? Want to review that more in-depth...

`src/Auth.Simple.Register`

- [ ] Better documentation
- [ ] `identifier`, `credential`, `register`, should all be in transactions, including looking to see if another user exists?
- [ ] Shouldn't there be a difference between an identifier and credential errors in the module? The errors in the module aren't that great...

`src/Auth.Simple.Login`

`src/Auth.Simple.Identifier`

`src/Auth.Simple.Credential`

`src/Auth.Simple.Token.Session`

`src/Auth.Simple.Identifier.Username`

`src/Auth.Simple.Credential.Stored`

`src/Auth.Simple.Credential.Code`

`src/Auth.Simple.Credential.Password`

`src/Auth.Simple.Credential.Password.PhcString`
