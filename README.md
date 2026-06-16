# gren-sqlite-auth

- Each module has a SQLite migration that is required to use its functionality, else interacting with the database will fail. You _must_ run the migration in the [User](Sqlite.Auth.User) module.
- All data is available for you to read, update, and manipulate outside of the confines of this package, given it lives in a SQLite database. This package primarily helps by ensuring authentication information is properly and safely saved and retrieved from the database and validated in situations when validation is necessary. Given all of the data this package operates on is available in an external SQLite database, it cannot prevent that data from being manipulated in ways that break the functionality of this package. In other words: use extreme caution when manipulating data this package depends on, otherwise it may break your authentication system!

## Why this package exists

The primary purpose of this package is to have a drop-in method for adding simple authentication primitives to your Gren application.

## What this package does

This package helps 

## What this package does _not_ do

This package is not meant to provide production-grade authencation. While care has gone into ensuring sensitive information is saved to the database properly to prevent against most attacks, it's highly recommended to review and deeply interrogate the implementations of such functionality before using it for critical systems.

## Quickstart

1. 
