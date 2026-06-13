# gren-sqlite-auth

- Each module has a SQLite migration that is required to use its functionality, else interacting with the database will fail. You _must_ run the migration in the [User](Sqlite.Auth.User) module.
- All data is available for you to read, update, and manipulate outside of the confines of this package, given it lives in a SQLite database. This package primarily helps by ensuring authentication information is properly and safely saved and retrieved from the database and validated in situations when validation is necessary.

## Why this package exists

## What this package does

## What this package does _not_ do

## Quickstart

1. 
