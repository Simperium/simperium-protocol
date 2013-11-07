# Simperium Streaming API

Simperium offers a streaming API that is accessible over a [Websocket][] or [SockJS][]. This document describes the messages a client can send and receive as well as how to implement syncing for a client.

[SockJS]: https://github.com/sockjs/sockjs-client
[Websocket]: http://www.websocket.org

## Contents

1. [Definitions of Terms](#definitions-of-terms)
2. [Connecting](#connecting)
3. [Streaming API](#streaming-api)
    1. [init (authorizing)](#authorizing-init)
    - [i (index)](#index-i)
    - [e (entity)](#entity-e)
    - [cv (change version)](#index-change-version-cv)
    - [c (changes)](#change-c)
    - [h (heartbeat)](#heartbeat-h)
    - [index (remote-index)](#remote-index)
    - [log (remote-logging)](#remote-logging)
4. [Syncing Bucket Objects](#syncing-bucket-objects)
    1. [Authorization](#authorization)
    - [First Sync](#first-sync)
    - [Requesting objects](#requesting-objects)
    - [Connecting with Existing Index](#connecting-with-existing-index)
    - [Receiving Remote Changes](#receiving-remote-changes)
    - [Sending Local Changes](#sending-local-changes)
    - [Server to Client Commands](#server-to-client-commands)


## Definitions of Terms


- **bucket** : a namespace for storing one an **entity**
- **entity** : a JSON serializable object that is stored in a **bucket**
- **index** : an array of hashes that contain an **entity**'s `key` and `v` (version)
- **cv** : An **index**'s last **change version** in an alphanumeric string: `7u37290aaddfkk`
- **ev** : An **e**nd **v**ersion of an entity in a [change operation `c`](#changec)
- **sv** : The **s**ource **v**ersion (the version the change applies to) of an entity in a [change operation `c`](#changec).
- **ccid** : An unique identifier for a specific change operation used in the `c` or [*command* message](#changec).

This assumes the client is starting with an empty **index**.

## Connecting

The client connects via websocket to `wss://api.simperium.com/sock/1/APP_ID/websocket` where `APP_ID` is replaced with the Simperium application id. Commands are sent over the websocket and can be prefixed with an integer which allows commands to be namespaced to a specific *channel* of communication to allow multiple buckets to be synced over the same socket connection.

## Streaming API

After connecting a client can send various commands to retrieve a bucket's objects and changes to facilitate syncing a local representation of a bucket with the one that exists on the server.

The available commands are:

- [init](#authorizing-init) - authorizes a connection to a bucket
- [i](#index-i) - requests an index of a bucket
- [e](#entity-e) - requests an entity's data for specified version
- [cv](#indexchangeversion-cv) - requests changes since a given index change version
- [c](#change-c) - send or receive a set of changes to perform on a bucket's objects
- [h](#heartbeat-h) - send and receive a heartbeat to maintain idle connection
- [index](#remote-index) - sent from server to client to request local client state
- [log](#remote-log) - sent from server to client to start/stop logging

### Authorizing: init

When a client is ready to connet to a user's bucket it sends the `init` command. The init command contains a JSON payload with the following key value pairs:

- **clientid** : a string that identifies the simperium client. Example: **simperium-andriod-1.0**
- **api** : the api version to use. Current: **1.1**
- **token** : a user's access token obtained via the auth api.
- **app_id** : a simperium app id. Example: **abusers-headset-123**
- **name** : the name of the bucket to use. Example: **notes**
- **library** : the library name of the connecting client: **simperium-android**
- **version** : the version of the connecting client library: **1.0**

Optionally, the init request can contain a command to be executed upon initialization:

- **cmd** : the command to executed after authorizing the channel: Example: **i::::500**

An example `init` command with a channel prefix `0`:

    0:init:{"api":1,"client_id":"android-1.0","token":"abc123","app_id":"abusers-headset","name":"notes"}

The Simperium server will respond with an `auth` command which will also contain either the authorized user's email address, or if authorization failed, a JSON error object.

Example failed auth:

    0:auth:{"msg":"Token invalid", "code":401}

Possible error codes:

- 401 - The token is affirmatively invalid, user needs new token.
- 500 - Other error, may be, bucket name is not valid, user has valid token but not permission for bucket, bucket limit reached, etc.

Example successful auth:

    0:auth:ender@example.com

After authorizing the client can now issue commands for the initialized bucket.

### Index: i

The index command -- `i` -- allows the client to receive all of the keys and corresponding version numbers for the objects stored in the bucket. The index command takes two parameters sperated by double colons `::`:

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
- **mark** : a cursor to be used in another **i:** request to fetch the next page of indexes for the bucket. **Note:** only present if there is more data to fetch from the index.

### Entity: e

Clients can request an entire entity at any version stored on the server. The `e` command takes one parameter which is an entity's `key` and `version` seperated by a dot `.`:

    0:e:keyname.1

The server will respond with the same name and version followed by a new line `\n` and the response for the entity. If the entity represented by that `key` and `version` does not exist, the response is a single question mark `?`:

    0:e:keyname.1
    ?

Otherwise the response will be a JSON payload. The entity's data is stored in the payload's `data` key:

    0:e:keyname.1
    {"data": {"1": 2, "0": 1, "2": "Three"}}


### Index Change Version: cv

To keep an existing index up to date, a client can request all changes since a specific *change version* or `cv`. The `cv` command takes one parameter: the *change version* to begin looking for bucket changes.

For example, the client may have stored an index locally that was received with change version `abc123`. The next time the client connects it will want to request all changes that may have happenned to the bucket while the client was disconnected. The client will send:

    0:cv:abc123

The server will look for all changes on the bucket since `cv` of `abc123`. If the `cv` does not exist for that bucket, the server will respond with:

    0:cv:?

Otherwise it will respond with a *change* `c` command that contains a JSON payload describing all of the changes that need to be applied to an index at change version `abc123` to match the current change version:

    0:c:[{"clientid": "sjs-2012121301-9af05b4e9a95132f614c", "id": "newobject", "o": "M", "v": {"new": {"o": "+", "v": "object"}}, "ev": 1, "cv": "511aa58737a401031d57db90", "ccids": ["3a5cbd2f0a71fca4933fff5a54d22b60"]}]

If the change version is up to date the server will respond with an empty `c` message:

    0:c:[]

### Change: c

To communicate changes to the bucket index clients and servers should send and respond to change messages: `c`. A change message contains a JSON payload which is an array of hashes that describe each change version to be applied to the index in order to bring it to a current state.

A change has these keys:

- **clientid** : the client that originally made the change
- **cv** : the change version of the index at the point of this change
- **ev** : the **e**nd **v**ersion for the entity after the change is applied
- **sv** : the **s**ource **v**ersion for the entity the change applies to. **Note:** not sent for new objects.
- **id** : the key of the entity this change applies to
- **o** : the type of operaton to perform
- **v** : the values to use for the operation
- **ccid** : a unique uuid to identify this specific change
- **d** : **Optional** the whole data object, see error cases **440**, **405** where this might be necessary

`change.ev` minus `change.sv` will usually be `1` but is not always the case.

Possible operations `o`:
- **M** : *modify* -- the operation is a modification to an entity
- **-** : *remove* -- remove the entity from the index

For modify operations the `v` key will contain an object diff compatible with [jsondiff][].

[jsondiff]: https://github.com/simperium/jsondiff

### Heartbeat: h

To keep a connection alive the client should send a heartbeat message while the connection is idle. This message *should not* be prefixed by a channel id since the heartbeat will maintain the connection for all channels.

The message takes one parameter, an integer that is incremented by the server and then sent back. Client sends:

    h:0

Server responds with:

    h:1

The client's next heartbeat message should increment the integer it received from the server. A heartbeat should be sent after 20 seconds of idle time and should expect an immediate response.

## Syncing Bucket Objects

A client needs to perform a specific set of operations to successfully keep its index synced with the remote version. We're assuming messages are sent over a channel with the prefix `0` and that the client is starting with an empty index.

### Authorization

To authorize access to a bucket the client will first need to obtain a user's access token using the [auth api][simperium-auth]. The client can then send an [`init`](#authorizinginit) command over the connection:

    0:init:{"name":"mybucket" ... }

The client should then wait for an `auth` response:

    0:auth:user@example.com

After a successful response the client should perform its first sync.

[simperium-auth]:https://simperium.com/docs/reference/http/#auth

### First Sync

Upon first connection to a bucket (e.g. the client has no data in its local index) a client will need to request the current index from the server and sync each of the objects. The client can request the index using the [`i` "index"](#index-i) command providing a limit for the page size.

Sending this message will request the bucket's latest index 100 items at a time:

    0:i::::100

The server will respond with an `i` message containing the JSON payload that represents a page of entity keys and versions:

    0:i:{"current": "5119dafb37a401031d47c0f7", "index": [{"id": "one", "v": 2}, ... ], "mark": "5119450b37a401031d3bfdb9"}

If there are more objects than fit in this page the server will send a cursor under the key `mark` that the client can use to request the next page. This command will request the next page of indexes from the server

    0:i::5119450b37a401031d3bfdb9::100

The client will know when it has received the entire index when it receives an `i` message without a `mark`.

After receiving a set of index data a client can begin requesting entity data from the server by requesting each `id.v` in the `index` key from the server. The client will want to store which `cv` they are syncing (the value under `current` in an `i` message).

### Requesting Objects

For each object in the `index` array of a `i` message, the client can request the entity's data using the [`e` "entity"](#entity-e). For example, this message asks the server to send `version` 2 of the entity stored at the key `qwerty`:

    0:e:qwerty.2

If the server has this entity and version it will respond with:

    0:e:qwerty.2
    {"data":{"message":"hello world"}}

The entity's data is stored in the `data` key of the JSON payload. The client should store both the data and the version locally so it can request changes for the entity in the future.

After storing all of the objects from an index request the client will have a synced copy of the bucket and can now start sending and apply changes.

### Connecting with Existing Index

After sending an `init` message, if a client already has local index data stored for a bucket it should send a [`cv` Change Version](#index-change-version-cv) message instead of an `i` message. Downloading an entire index of data would be wasteful.

The client should have stored the current *change version* for the index so it can ask the server for all changes since that version in order to catch up.

    0:cv:5119dafb37a401031d47c0f7

If the server knows about this change version for the connected bucket it will respond with the changes necessary to transform the local index to match the remote one:

    0:c:[{"clientid": "sjs-2012121301-9af05b4e9a95132f614c", "id": "newobject", "o": "M", "v": {"new": {"o": "+", "v": "object"}}, "ev": 1, "cv": "511aa58737a401031d57db90", "ccids": ["3a5cbd2f0a71fca4933fff5a54d22b60"]}]

If the server doesn't have the requested *change version* it will send this `cv` message:

    0:cv:?

At which point the client will need to [reload the index](#requesting-objects). To save space the server starts aggregating older change versions so a single change version is not permanently stored forever.

### Receiving Remote Changes

A client will receive remote changes from the server either by explicitly asking for them ([using the `cv`](#index-change-version-cv)) or simply by being connected when a server receives a [`c` change](#change-c) command. A remote change message will contain a JSON payload that is an array of changes and information about those changes (corresponding `ccid`s and an index `cv` for the changes). An example incoming `c` message representing a single change:

    0:c:[{"clientid": "sjs-2012121301-9af05b4e9a95132f614c", "id": "newobject", "o": "M", "v": {"new": {"o": "+", "v": "object"}}, "ev": 1, "cv": "511aa58737a401031d57db90", "ccids": ["3a5cbd2f0a71fca4933fff5a54d22b60"]}]

To apply these changes a client will want to loop through each change and perform the operation described in the change object:

  1. Get the local entity using `change.id` as the key
      - If client doesn't have the entity, request it using [e](#entity-e) command
  2. Confirm that the local entity version matches the `change.sv`
      - If they don't match, [reload](#reloading-an-object) the object
  3. Apply the change:
      - If `change.o` is `-` remove the entity from the local store
      - If `change.o` is `M` apply `change.v` using [jsondiff][]
      - Set the entity's version to `change.ev`
  4. Save the `change.cv` for the index

#### Error cases for receiving changes

- Applying the change:
  If the client cannot apply the change, it needs to re-load the object

- Change for missing entity:
  If a change is received referencing an entity that doesn't exist locally (and the change isn't creating the entity), client needs to re-load the object

#### Reloading an object

In some cases the client needs to reload an object, when it does so it should do the following:

  1. Issue a [`e` "entity"](#entity-e) command
  2. Upon receipt of the object, check if there are local client modifications
  3. If there are no local modifications, update the local index
  4. If there are local modifications, a local transform of the modifications vs the diff between the existing local index version of the object and the just downloaded version.

### Sending Local Changes

When the client is ready to update an object's representation in the remote datastore it needs to generate the diff and write a [`c` command to the server](#change-c).

After sending the change, the client should flag the change as sent but not yet "acknowledged" until it receives a `c` response from the server containing the `ccid` that was sent. If the change was successful the client can consider the local representation as up to date.

In some cases the server will respond with a error response if it could not apply the change. An example error response:

    c:[{"clientid":"offending-client","id":"object-key","error":"401","ccids":["abcdef123456"]}]


### Server to Client commands

The server may ask the client at any time for some data

- **index** - response should be the current local state of a specific bucket
- **log** - sent to client to turn on/off logging and indicate logging level

#### Remote index

Server may send a command requesting the index for a particular bucket, this command will be sent on a particular channel that is already open

Example request from server on channel 0:

    0:index

Response from client should follow:

    0:index:{ current: <cv>, index: [ {id: <eid>, v: <version>}, ... ], pending: [ { id: <eid>, sv: <version>, ccid: <ccid> }, ... ], extra: { ? } }


#### Remote logging

Server may ask the client to start sending log messages from the client, this request will be sent without a channel prefix, responses should also be sent without a prefix.

We currently define 3 states of logging:

    **0** : Client should stop sending remote logging messages
    **1** : Client should send logging messages
    **2** : Client should send "verbose" logging messages

The response from the client should be a json object containing at least a "log" field. The value from this field will be logged.

Example request from server (requesting normal logging):

    log:1

Example response:

    log:{"log":"<some log message>"}


#### Error Responses

Potential error responses:

- **400** : invalid id or invalid schema - change request did not conform to required fields
- **401** : invalid permission
- **404** : not found - object key or key version not found
- **405** : bad version
- **409** : duplicate change
- **412** : empty change
- **413** : document too large
- **440** : invalid diff (wrong key, bad delta), invalid schema
- **5xx** : internal server error

Handling Errors:

**400** : If it was an invalid id, changing the id could make the call succeed. If it was a schema violation, then correction will depend on the schema. If client cannot tell, then do not re-send since unless something changes, 400 will be sent every time.

**401** : Grab a new authentication token (or possible you just don't have access to that document).

**404** : Client is referencing an object that does not exist on server. If client is insistent, then changing the diff such that it is creating the object instead should make this succeed.

**405** : Bad version, client referencing a wrong or missing `sv`. This is a potential data loss scenario: server may not have enough history to resolve conflicts. Client has two options:
  - re-load the entity (via `e` command) then overwrite the local changes
  - send a change with full object (which will overwrite remote changes, history will still be available) referencing the current `sv`

**409** : Duplicate change, client can safely throw away the change it is attempting to send

**412** : Empty change, nothing was changed on the server, client can ignore (and stop sending change).

**413** : Nothing to do except reduce the size of the object

**440** : Server could not apply diff, resend the change with additional parameter `d` that contains the whole JSON data object. Current known scenarios where this could happen:
  - Client generated an invalid diff (some old versions of iOS library)
  - Client is sending string diffs and using a different encoding than server



