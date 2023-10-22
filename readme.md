# Inspiration

Most Key/Value stores use the term bucket.

From [NATS](https://docs.nats.io/using-nats/developer/develop_jetstream/kv)

```
Creating, and deleting KV buckets
You can create as many independent key/value store instance, called 'buckets', as you need. Buckets are typically created, purged or deleted administratively (e.g. using the nats CLI tool),
...
```

or like this article from [DZone](https://dzone.com/articles/bucketkeyvalue-paradigm)

```
Bucket/Key/Value Paradigm as a Replacement For Relational DBMS
```

Naming items in schemas and code is never easy so a well known github 44.7k star project has been used as inspiration.

## [etcd](https://etcd.io/) [kv.proto](https://github.com/etcd-io/etcd/blob/main/api/mvccpb/kv.proto)

### KeyValue

```proto
message KeyValue {
  // key is the key in bytes. An empty key is not allowed.
  bytes key = 1;
  // create_revision is the revision of last creation on this key.
  int64 create_revision = 2;
  // mod_revision is the revision of last modification on this key.
  int64 mod_revision = 3;
  // version is the version of the key. A deletion resets
  // the version to zero and any modification of the key
  // increases its version.
  int64 version = 4;
  // value is the value held by the key, in bytes.
  bytes value = 5;
  // lease is the ID of the lease that attached to key.
  // When the attached lease expires, the key will be deleted.
  // If lease is 0, then no lease is attached to the key.
  int64 lease = 6;
}
```

### Event

```proto
message Event {
  enum EventType {
    PUT = 0;
    DELETE = 1;
  }
  // type is the kind of event. If type is a PUT, it indicates
  // new data has been stored to the key. If type is a DELETE,
  // it indicates the key was deleted.
  EventType type = 1;
  // kv holds the KeyValue for the event.
  // A PUT event contains current kv pair.
  // A PUT event with kv.Version=1 indicates the creation of a key.
  // A DELETE/EXPIRE event contains the deleted key with
  // its modification revision set to the revision of deletion.
  KeyValue kv = 2;

  // prev_kv holds the key-value pair before the event happens.
  KeyValue prev_kv = 3;
}
```

## [patch.proto](proto/patch.proto)

Revision and version from the KeyValue message have been put together into a Meta message.

```proto
message Meta {
    int64 create_revision = 1;
    int64 mod_revision = 2;
    int64 version = 3;
}
```

The EventType is put into (patch) Type

```proto
enum Type {
    PUT = 0;
    DELETE = 1;
}
```

This is combined into (patch) Item

```proto
message Item {
    Type type = 1;
    bytes key = 2;
    bytes value = 3;
    Meta meta = 4;
}
```

The Item message are used in

```proto
message PullResponse {
    ...
    repeated Item list = 3;
    ...
}
```

and

```proto
message PushRequest {
    ...
    repeated Item list = 3;
    ...
}
```

(todo? rename list => items)

The patch service is defined with the pull and push methods with a set of top level request and response messages.

```proto
service Service {
    rpc Pull(PullRequest) returns (PullResponse) {}
    rpc Push(PushRequest) returns (PushResponse) {}
}
```

Common for all top level messages is the Query message.

```proto
message Query {
    bytes path = 1;
    int32 limit = 2;
    int32 timeout = 3;
}
```

The reason for having it both in the request and response is to enable handshake about the limit (todo discuss this).

The path member is the path to a bucket like the path to a file.

## [management.proto](proto/management.proto)

This is the default schema for the Cursor message but might be replaced as long as it can support the same functionality.

A Meta message similar to the patch version is used in the Cursor together with the session (key).

```proto
message Cursor {
    bytes session = 1;
    Meta meta = 2;
}
```
