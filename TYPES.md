# Simperium Data Types

There are certain types of data that Simperium uses which have specific constraints. Check here for those data types and constraints.

## Basic types

| type | min. length | max length | allowable values | notes |
|---|:-:|:-:|---|---|
| **app id** | | | `a-z` `A-Z` `0-9` `.` `_` `-` | case insensitive |
| **bucket name** | 1 | 64 | `a-z` `A-Z` `0-9` `.` `_` `-` `%` | |
| **ccid** | 1 | 256 | `a-f` `A-F` `0-9` | [type 4 (random) UUID](https://tools.ietf.org/html/rfc4122) in hex |
| **change type** | | | `M` `-` `DROP` `EMPTY` | see _change types_ below|
| **client id** | 1 | 256 | | |
| **entity key** | 1 | 256 | `a-z` `A-Z` `0-9` `.` `_` `-` `%` `@` | |
| **entity version** | | | integer | |
