# derby-storage-protocol
Derby network storage protocol specification

The Derby protocol is a very simple protocol that centers around the searching, publishing, and updating of pointers.

The protocol uses a similar approach to management of data and client\server architecture as Nostr.

Please note,The Derby protocol is **not** compatible with the Nostr protocol.

# Network

The Derby protocol uses WebSockets for connectivity. Storage nodes are referenced by ws:// and wss://

The protocol is plain text only. All data is transmitted using Base64, which has a tradeoff on increased transfer size and conversion time.

# Pointers

The Derby protocol uses a data structure called a pointer to manage and reference data. The pointer object is a JSON object with fixed fields.

A pointer looks like the following:

```
{
  id: SHA-256 Hash of the Pointer Object <String>==,
  pubkey: 256-bit hex key <String>,
  timestamp: Unix time in seconds <Integer>,
  pointerhash: SHA-256 Hash of the referenced data <String>,
  size: Size of referenced data in bytes <Integer>,
  nonce: Arbitrary integer value <Integer>,
  signature: Schnorr signature of id using private key <String>
}
```

An example of a real world pointer:
```
{
  "id":"c868b2defabe0683b5426fd66318db1beac1c6af7143f75f389926ac28a827f7",
  "pubkey":"1a305a7ae5d63329fc3597155521638ff1c5d989285b5a7be275e38826f12885",
  "timestamp":1699213847,
  "pointerhash":"f323fe7ecdacc0bba46a8bd70ea61f2622297b40b0a93ab2beabd3a03a2a7bbd",
  "size":5000000,
  "nonce":513029,
  "signature":"37e721ad91c2323f4b73ccc4e2c006632b348c6b99b5f372b796daab3b75d1062e727687a41ac579d088d1db2121015a7df6cf2e049024d199de880d894e81ac"
}
```

The ID is derived from the specific order of the JSON as:
* pubkey
* timestamp
* pointerhash
* size
* nonce

Also note that the nonce is used for pointer deletion rules and the values 0-9 are reserved for deletion pointers.

# Server Commands

Storage nodes have 5 actions that can be taken using the base protocol:
* Publish a pointer with Data
* Publish a pointer referencing existing data
* Update an existing pointer
* Delete a pointer
* Download data referencing a pointer id

## Publishing a pointer

A pointer can be published by using the following command:

`["POINTER", $POINTER_OBJECT, "PUBLISH", {$BASE64DATA}]`

### Data

Data is represented in a Base64 string format. 

If a new data is being uploaded to the server, the data must be present. Data can be omitted if the data already exists on the server, referenced by another pointer.

### Pointer publishing rules

A pointer must adhere to the following rules to be accepted:
* Pointer id must be a valid SHA-256 hash of the pointer data.
* Timestamp must be within a storage node's defined time delta (i.e. within +/- 300 seconds as defined by a storage node)
* Only one pointer hash can be referenced per public key. Otherwise the pointer will be updated and the old one will be removed.
* Pointer hash must be the SHA-256 hash of the raw binary data, not the Base64 data.
* Size must be of the raw binary data, not the Base64 data.
* Signature must validate against the id and public key.

### Valid Response

A response for a successful pointer published is:

`[“OK”, $ID, $POINTERHASH]`

## Replacing a pointer

Replacing a pointer is uses the same command as publishing a pointer.

You can replace a pointer by providing a new pointer with the same pubkey, pointerhash, and size with an updated timestamp.

If the same data that is already on the server is also uploaded with the pointer, it will be ignored and the pointer will be updated.

A response to an updated pointer will be identical to publishing a new pointer:

`[“OK”, $ID, $POINTERHASH]`

# Removing a pointer

Removing a pointer requires generating a new pointer and using a delete command:

`["POINTER", $POINTER_OBJECT, "DELETE"]`

The purpose of generating a new valid pointer is to prove to the storage node that the key holder is able to generate a new signed object. This prevents an attacker from using an older signed pointer to delete a pointer.

Two rules for a deletion pointer:
* Timestamp must be newer than the existing pointer
* Timestamp can be +/- 300 seconds off of system time
* Nonce must be different than the existing pointer
* Nonce must be between 0 and 9

**Note that normal pointers can have a nonce value of 0 through 9, but it's not recommended since an attacker can take a recently published pointer and use that to delete pointers on another relay.**

## Valid response

A response for a successful deleted pointer is:

`[“OK”, $DELETION_POINTER_ID , $DELETED_POINTERID]`

# Pointer requests

The Derby protocol allows clients to search for pointers based on various criteria.

Data cannot be retrieved as part of a query.

A request looks like the following:

`[“REQUEST”,$REQID, {ids:[$ids],owners:[$owners],olderthan:$timestamp,since:$timestamp,pointerhashes:[$pointerhashes],sizeis:$size,sizelargerthan:$size,sizesmallerthan:$size,limit:$limit}]`

$REQID is an arbitrary string referencing the query.

* ids - Array of pointer ids
* owners - Array of pubkeys
* olderthan - Unix time in seconds for pointers created before the specified time, exclusive
* since - Unix time in seconds for pointers created since the specified time, inclusive
* pointerhashes - Array of data referenced by hash
* sizeis - Pointers with a specified size in bytes
* sizelargerthan - Pointers referencing data with a size larger than the spcified value in bytes, exclusive
* sizesmallerthan - Pointers referencing data with a size smaller than the specified value in bytes, exclusive
* limit - Number of pointers to return from query. 0 will be ignored and all pointers up to hard limit of the storage node (1000 pointers) will be retrieved.

A request response returns the following:

`["POINTER", $REQID, [$POINTER[0], $POINTER[1], ..., $POINTER[N]]]`


`["REQEND",$REQID]`

An request without any results will return an empty array:

`["POINTER", $REQID, []]`

## Downloading Data

Data can be downloaded by referencing the pointer id that references a particular block of data.

Data is retrieved in Base64 format.

A request looks like the following:

`[“REQDATA”,$POINTERID]`

If the pointer and data exists a succesful response will be:

`[“DATAOK”,$POINTERID,$POINTERHASH,$BASE64DATA]`

The SHA-256 hash of the raw data should match the $POINTERHASH.

# Errors

The following section will discuss errors including defined error codes and expected error responses for each of the server commands.

Error codes are formatted in the following way:

`[“ERROR”, $ERRORCODE, $CONTEXT, $MESSAGE]`

* $ERRORCODE - Numeric Error code stating the class of error (i.e. Invalid Pointer).
* $CONTEXT - If applicable, references the relevant id based on the error (Pointer id if an error on pointer creation, request ID if a bad request).
* $MESSAGE - String error message to provide additional human readable information.

## Error Types

### 0x00 - Invalid request

Error 0 is a generic invalid request error. This will be sent if the protocol request is malformed JSON or expected number of inputs are not satisified.

There will be no context applied here since this is a general error and there would be no ids to read from.

### 0x01 - Invalid Values

Error 0x01 are for properly formatted requests with bad values. An example is a command that expects 3 values and only gets 2.

The context will be blank here as there will be no id to reference.

### 0x02 - Invalid Command

Error 0x02 is similar to Error 0x01 but is specifically for a command the storage node does not recognize. An example would be to send something like:

`["POINTER",$POINTER_OBJECT,"ADD"]`

The storage node does not recognize "ADD" and will send a 0x02 error.

The context will be blank as there will be no id to reference.

### 0x03 - Invalid Query

A malformed query object will send back this error. 

The context will be the request ID of the original "REQUEST" command

### 0x04 - Invalid Pointer

Error 0x04 encompasses all issues around adding, removing, or updating pointers. This includes:
* Publishing a pointer that fails ID and/or Verification
* Submitting that fails the timestamp delta restrictions
* Malformed JSON within the pointer
* Requesting data using a pointer id that does not exist

The context is the pointer ID attempted to be created or referenced.

The error message will provide specifics on the error.

### 0x05 - Invalid Data Hash

Error 0x05 is more specific around a mismatch between the reported pointerhash and the actual SHA-256 hash of the raw binary.

The context is the pointer id being submitted.

### 0x06 - Invalid Size

Error 0x06 is more specific around a size mismatch between the reported size in the pointer object and the actual size of the raw binary.

The context is the pointer id being submitted.

### 0x07 - Invalid Deletion Pointer

Error 0x07 is a more specific error related to the deletion pointer rules. If a pointer fails the timestamp or nonce rules it will be sent instead of the more general 0x04 error.

The context will be the deletion pointer id.

### 0x08 - Error retrieving data

Error 0x08 is a general error if there is a backend error retrieving data using the "DATAREQ" command. This could be because the storage node is unable to read from disk or the blob may be unavailable for any other reason

The context will be the referenced pointer id.

## 0x09 - Error retrieving pointers

Error 0x09 is a more specific error than 0x04 as the storage node may be unable to retrieve the pointer from the database for an underlying systems reason

***Additional Errors TBD***


