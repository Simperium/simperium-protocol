# Simperium Bucket Options

Simperium buckets have multiple options that control the behavior of
objects stored in them.

A bucket's options is stored in a JSON object (itself in a specially named
simperium bucket). Default options can also be stored, to determine a specific
buckets options, the default options object (if it exists) is retrieved, then
the bucket's own options object (if it exists) is applied on top (replacing
any fields that were defined in the default object).

## Retrieving and updating options

Options are stored in the bucket named `__options__` as Simperium objects. The
key of the object corresponds to the bucket for which the options apply. The
options bucket must be accessed using a token generated from an API key with admin
privileges.

Example: The options for the bucket `testbucket` for the app `sample-app` can be
retrieved using the standard HTTP API:

    GET https://api.simperium.com/1/sample-app/__options__/i/testbucket

Since this is a normal simperium bucket (the only distinguishing feature being
the name), versioning is supported.

Updates to options objects are handled similarly to other simperium objects:

    POST https://api.simperium.com/1/sample-app/__options__/i/testbucket

## Option fields

- **capacity** : integer, default: None. If set, this defines the maximum number
  of items in this bucket per user. If sorting is not set, then the least
  recently changed items are removed when inserting new items past the capacity
  limit.

- **version_max** : integer, default: 60
- **version10_max** : integer, default: 100

  These settings control how many versions are stored. *version_max* refers to
  the number of contiguous versions. If *version_max* is 50, once we
  make the 51st update, version 1 will no longer be available. *version10_max*
  refers to the number of every 10th version we store after the *version_max*
  limit is reached. For example: if *version_max* is 50 and *version10_max* is
  2, and we make 100 updates to an update such that it is currently at version
  100, then versions 51-100 will be available, then, 41, and 31.

- **schema** : Object, default: None

- **delete-schema** : Object, default: None






