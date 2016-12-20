# Simperium Data Types

There are certain types of data that Simperium uses which have specific constraints. Check here for those data types and constraints.

## Basic types

| type | min. length | max length | allowable values | notes |
|---|:-:|:-:|---|---|
| **app id** | | | `a-z` `A-Z` `0-9` `.` `_` `-` | case insensitive |
| **bucket name** | 1 | 64 | `a-z` `A-Z` `0-9` `.` `_` `-` `%` | |
| **ccid** | 1 | 256 | `a-f` `A-F` `0-9` | [type 4 (random) UUID](https://tools.ietf.org/html/rfc4122) in hex |
| **change type** | | | `M` `-` `EMPTY` `DROP` | see _change types_ below|
| **client id** | 1 | 256 | | |
| **entity key** | 1 | 256 | `a-z` `A-Z` `0-9` `.` `_` `-` `%` `@` | |
| **entity version** | | | integer | |

## Change types

There are four primary change types available to the Simperium protocol.
Of these, `DROP` is special in that it requires administrative privileges on the bucket to be able to successfully execute.

### Modify entity - `M`

The _modify_ change type means that we intend on changing the value stored for an entity which already exists in the bucket.

### Delete entity - `-`

The _delete_ change type means that we intend on removing an entity and its stored value from the bucket.
Once an entity is deleted it may or may not persist in the bucket.
If it _does_ persist, it will be flagged as deleted and have its own entity version associated with the removal.
If it _does not_ persist, it will be gone completely.
Whether or not it persists depends on the retention policy set for the bucket.
Only the most-recent version of an entity may be deleted.

**May throw**
 - `BAD_VERSION` if newer versions of the entity exist on the server than the version specified in the _delete_ request.
 - `INVALID_PERMISSION` if the requesting user does not have permission to write to the specified entity.
 - `MISSING_ENTITY` if the specified entity/version pair does not exist on the server

### Empty bucket - `EMPTY`

The _empty_ change type means that we intend on clearing out all of the entities in a specified bucket.
This process is different than the _delete_ change type in that it will not flag the entities as deleted and create versions for the deletion.
There is no retention when emptying a bucket.

### Drop bucket - `DROP`

The _drop_ change type means that we intend on removing the entire specified bucket from the server.
There is no retention when dropping a bucket.
This operation requires administrative privileges on the specified bucket.

**May throw**
 - `INVALID_PERMISSION` if the requesting user does not have administrative privileges on the specified bucket.
