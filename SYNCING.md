# Syncing

How to sync.

## Definitions


- **bucket** : a namespace for storing one an **entity**
- **entity** : a JSON serializable object that is stored in a **bucket**
- **index** : an array of hashes that contain an **entity**'s `key` and `v` (version)
- **ccid** or **cv** : An **index**'s last change version in an alphanumeric string: `7u37290aaddfkk`

This assumes the client is starting with an empty **index**.

## Connecting

The client connects via websocket to `wss://api.simperium.com/sock/websocket`. Commands are sent over the websocket and can be prefixed with an integer which allows commands to be namespaced to a specific *channel* of communication to allow multiple bucketes to be synced over the same socket connection.

### Authorizing: init

When a client is ready to connet to a user's bucket it sends the `init` command. The init command contains a JSON payload with the following key value pairs: 

- **clientid** : a string that identifies the simperium client. Example: **simperium-andriod-1.0**
- **api** : the api version to use. Example: **1**
- **token** : a user's access token obtained via the auth api.
- **app_id** : a simperium app id. Example: **abusers-headset-123**
- **name** : the name of the bucket to use. Example: **notes**

Optionally, the init request can contain a command to be ran upon initialization:

- **cmd** : the command to run after authorizing the channel: Example: **i::::500**

An example `init` command with a channel prefix `0`:

    0:init:{"api":1,"client_id":"android-1.0","token":"abc123","app_id":"abusers-headset","name":"notes"}

The Simperium server will respond with an `auth` command which will also contain either the authorized user's email address, or if authorization failed, the string `expired`.

Example failed auth:

    0:auth:expired
    
Example successful auth:
    
    0:auth:ender@example.com

After authorizing the client can now issue commands for the initialized bucket.

### Index: i

The index command -- `i` -- allows the client to receive all of the keys and corresponding version numbers for the entities stored in the bucket. The index command takes two parameters sperated by double colons `::`:

- **mark** : *optional* a cursor that indicates where in the index you are requesting
- **limit** : the maximum number of items to return (optional? is there a default on the server side?)

When connecting for the first time with an empty index, the client should issue an `i` command with a sane limit. The following example requests the bucket's index with a page size of 500 items.

    0:i::::500
    
The server will respond with the index results in a JSON payload prefixed with `i:`. Example response:

    0:i:{"current": "5119dafb37a401031d47c0f7", "index": [{"id": "one", "v": 2}, ... ], "mark": "5119450b37a401031d3bfdb9"}
    
The JSON payload contains these keys:

- **current** : the bucket's current *change version* or `cv`/`ccid`. Clients should store this for future requests to indicate to the server which version of the index they have.
- **index** : an array of hashes that contain:
  - **id** : the entity's unique key name
  - **v** : the entity's version number
- **mark** : a cursor to be used in another **i:** request to fetch the next page of indexes for the bucket

### Entity: e

Clients can request an entire entity at any version stored on the server. The `e` command takes one parameter which is an entity's `key` and `version` seperated by a dot `.`:

    0:e:keyname.1
    
The server will respond with the same name and version followed by a new line `\n` and the response for the entity. If the entity represented by that `key` and `version` does not exist, the response is a single question mark `?`:

    0:e:keyname.1
    ?
    
Otherwise the response will be a JSON payload. The entity's data is stored in the payload's `data` key:

    0:e:keyname.1
    {"data": {"1": 2, "0": 1, "2": "Three"}}


