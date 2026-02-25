# Protostar 

Protostar is an alternative Concrete Syntax Structure and configuration language for the Protobuf data serialization format, intended to:

* More closely represent a protobuf AST at a per-file level, encoding messages and service names at the filesystem level, and structuring imports there too

* Provide a more developer-friendly interface for defining protobuf messages, choosing a subset of the protobuf format as a compilation format and introducing some new genericized types

* Provide a system for configuring protobuf services and resolving their types modularly with file lookups

* Serve as a kind of intermediate format for other languages' (eg Golang, ideally more) to use for automated wrapping with protobuf services: .go -> pkg or ast -> exportservice.protostar -> .go wrapper, .go main linker, exportservice.proto -> compiled grpc service.

In Accretional's type/API system it's the format that we think protobuf APIs should be represented with; it's our general CST bridge between existing software, protobuf, and our own abstractions (see proto-type for more details).

# Motivation / Rant

Suppose we want to define a protobuf service called FooService. Normally we need to define its request and response message formats in the same file, import packages there, do some others stuff, and that file might be named something like foo.proto

<details>
  <summary>Click to expand boilerplate</summary>
  
```proto

syntax = "proto3";

package foo.v1;

option go_package = "github.com/example/foo/v1;foov1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/any.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";

enum Status {
  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE      = 1;
  STATUS_INACTIVE    = 2;
  STATUS_SUSPENDED   = 3;
}

message Foo {
  reserved 15; // why is this at the top of the file
  reserved "legacy_name", "old_description"; // is this the same or different from reserved field nums

  // look I can count to a 100
  string      name        = 1;
  int32       version     = 2;
  int64       size_bytes  = 3;
  uint32      retry_count = 4;
  double      score       = 5;
  float       weight      = 6;
  bool        enabled     = 7;
  bytes       checksum    = 8;
  fixed64     fingerprint = 9;
  Status   status   = 10;
  // lol oops forgot 11
  // wait why are all these large numbers defined and 11 is not reserved. what does that mean

  message Metadata {
    string created_by = 1;
    string region     = 2;

    message AuditEntry {
      string actor  = 1;
      string action = 2;
      // why am I always importing and spelling out the same long type names I use all the time
      google.protobuf.Timestamp performed_at = 3;
    }

    repeated AuditEntry audit_log = 3;
  }

  Metadata metadata = 17;

  google.protobuf.Timestamp  created_at  = 18;
  google.protobuf.Timestamp  updated_at  = 19;
  google.protobuf.Duration   ttl         = 20;
  google.protobuf.Any        extra       = 21;

  // sign of a well designed language: don't allow rpcs between primitive types because only messages are forwards and backwards compatible
  // then in case does need to just send a string, provide default wrapper types that can't easily be modified
  google.protobuf.StringValue  display_name = 22;
  google.protobuf.Int32Value   max_retries  = 23;
  google.protobuf.BoolValue    archived     = 24;

  repeated string tags    = 25;
  // We lack the technology to operate on such complex structures at the rpc interface level
  // Nobody knows exactly what "repeated" means, but if you thought "repeated Bar" was a type, lol, idiot
  repeated Bar    items   = 26;

  // what if we had generics but only one and with weird restrictions
  map<string, string>   labels      = 27;
  map<string, int32>    counters    = 28;
  map<int32, Bar>       bar_by_code = 29;

  // Same as a union type but worse
  oneof source {
    string       url           = 30;
    bytes        inline_data   = 31;
    FileRef      file_ref      = 32;
    DatabaseRef  database_ref  = 33;
  }

  // Can't wait to count to even higher numbers, gonna leave a little teaser
  // NEXT: 34
}

message Bar {
  string id    = 1;
  string label = 2;
  double value = 3;
  repeated string attributes = 4;
}

message FileRef {
  string bucket = 1;
  string path   = 2;
  string etag   = 3;
}

message DatabaseRef {
  string connection_string = 1;
  string table             = 2;
  string query             = 3;
}

message CronSchedule {
  string expression = 1;  // e.g. "0 */5 * * *"
  string timezone   = 2;
}

message IntervalSchedule {
  google.protobuf.Duration every  = 1;
  google.protobuf.Duration jitter = 2;
}

// Important language convention: every rpc must have its own unique input-output pair types, but they cannot be defined at the rpc definition in case someone else needs to use this type that explicitly must only ever be used by this rpc
// Even more important language convention: every service must implement its own incomplete version of CRUD with inconsistent semantics 
message CreateFooRequest {
  Foo foo            = 1;
  string idempotency_key = 2;
}

message CreateFooResponse {
  Foo foo = 1;
}

message GetFooRequest {
  string id = 1;
}

message GetFooResponse {
  Foo foo = 1;
}

message UpdateFooRequest {
  Foo foo = 1;
  // Field mask for partial updates (standard pattern, defeats the purpose of the language tho, why are we writing out all these numbers again?)
  repeated string update_mask = 2;
}

message UpdateFooResponse {
  Foo foo = 1;
}

message DeleteFooRequest {
  string id = 1;
}

// Uses google.protobuf.Empty for the delete response

message ListFoosRequest {
  int32  page_size   = 1;
  string page_token  = 2;

  Status status_filter  = 3;
  string order_by       = 4;
}

message ListFoosResponse {
  repeated Foo foos       = 1;
  string next_page_token  = 2;
  int32  total_count      = 3;
}

enum Priority {
  PRIORITY_UNSPECIFIED = 0;
  PRIORITY_LOW         = 1;
  PRIORITY_MEDIUM      = 2;
  PRIORITY_HIGH        = 3;
  PRIORITY_CRITICAL    = 4;
}

// Collapses the service-level wave function between not ready yet and deprecated
// IMPORTANT: defy expectations and do not make all kinds of other rpcs batchable in the request
// ALSO: the response must include useless redundant fields in the response because LLMs don't know that they're already becoming their parents
message BatchProcessRequest {
  repeated string ids = 1;
  Priority priority   = 2;
}

message BatchProcessResponse {
  int32 succeeded = 1;
  int32 failed    = 2;

  message Failure {
    string id     = 1;
    string reason = 2;
  }

  repeated Failure failures = 3;
}

// boilerplate builds character
service FooService {
  rpc CreateFoo (CreateFooRequest)  returns (CreateFooResponse);
  rpc GetFoo    (GetFooRequest)     returns (GetFooResponse);
  rpc UpdateFoo (UpdateFooRequest)  returns (UpdateFooResponse);
  rpc DeleteFoo (DeleteFooRequest)  returns (google.protobuf.Empty);
  rpc ListFoos  (ListFoosRequest)   returns (ListFoosResponse);

  rpc BatchProcess (BatchProcessRequest) returns (BatchProcessResponse);
}

```
</details>

This is quite a lot of boilerplate. I'd like to bring several design flaws in .proto to your attention, because once you see them, and how often they constitute everything missing in an API + everything boilerplating and annoying in them, you'll see them forever more.

## google.protobuf.X

You basically always have to import at least a few of these for anything non-trivial, often the usual suspects. Unfortunately for you the CLI tooling wants you to install them separately from other proto tools and nobody makes it easy even though you basically always needs this.

These need to be moved into the language or at least given shorter import/type names. Also there should maybe be more but we'll get to that.

## Counting to a hundred

There is a big problem with the .proto field numbering system's tradeoffs. First off, these are sparsely encoded (but do use a varint type that limits itself to O(logn) the max field num) numeric ids. Why? Well, if someone deprecates or renames a field.

Ok, if they deprecate or rename a field, can't we just do so by keeping it in the same position, and use a dense sequential ordering system with holes for deprecated fields? Then I can just encode everything positionally and never have to do all this counting?

Proto says, well sure, but

* what if I want to swap the position of two fields in the file but keep the messages in the same format

* once the field is removed am I just suppoesd to keep that hole there forever?

* what if I just want there to be a hole at that position, huh?

* I need to be able to new fields anywhere I want relative to other fields

Ok well, you can do 2 and 3 positionally too, you just redefine the field. 1 and 4 are purely cosmetic changes of the .proto file. It's such a self-own to optimize for "potentially wanting to change Foo foo; Bar bar; to Bar bar; Foo foo; because it makes the schema cleaner over CONSTANTLY COUNTING THINGS AND HAVING TO KEEP TRACK OF ALL THESE DAMN NUMBERS IN THE MIDDLE. THE THING COMPUTERS DO FOR ME, WITHOUT LEAVING RANDOM NUMBERS IN THE MIDDLE UNUSED.

We will abolish the field numbers at the interface level.

## Wrapper types

These make no sense: string can't be used as rpc input/output types because those need to be messages which can't be primitive types to enforce forward compatibility via extensible messages. Oh well if you want a workaround just a StringValue which is even less extensible than if you just let me do rpc Method(string) -> rpc Method(string, int) like a normal type system. If I know enough to know what a morphism is and why it's nice defining things that way, I also know enough to know what a product type is and why anonymous tuples/inline types make life so much more pleasant.

Wrapper types are such stark reminders of could have been that they feel almost like a cry for help. You know what would be even better? Wrappers that implement product types. Oh maybe coproduct types too while you're at it

## Optional

I don't know what this even means any more in proto3 but I'm pretty sure I really do just want an optional type. Maybe that bit can work second shifts in enums' first fields.

## Repeated and Map

It's literally cruel that Map taunts generics and inspire hope in encoding more complex types directly at the interface level, and perhaps have them treated as types unto themselves, and then see repeated starting at you and laughing saying "Map only gave you that syntax as a joke".

Do you know what repeated is? Repeated is part of the FieldDescriptor proto, but it's variable-length so encoding it requires treating every repeated Foo like a List<Foo> in the same way, but no seriously it's just a field configuration option.

Imagine the world that could have been: List<Foo>, legitimized as a rightful type and accepted as rpc ListFoo(ListFooRequest) returns List<Foo>; or rpc StoreTable(List<Foo>) returns (YeahThatsRightGenericTables); or waking up in a bed only to sigh with relief that  MultiFoo { repeated Foo foos = 1; } x 20 Foo was just a strange dream.

We are going to implement Collection<Foo>, and I want to reiterate how important this is by asking you to [check out Google's API design guide](https://google.aip.dev/121) if you haven't already.

Resource oriented design, what is it? A million and one rules for CRUD servers wrapping "resources" for stateful storage. [Look at this](https://google.aip.dev/133), do you understand how contradictory it is to have this level of human-driven standards specification and enforcement for a data/transport format with its own type system? 

**If Collection<T> were respected as a first-class type, resource oriented design an all these rules/principles could be almost entirely simplified into a generic CRUD ORM for Collection<T>.** That alone is a massive win for proto, no more BatchFooRequest.

You could also, idk, do some pretty convenient stream/tabular/relational processing if you could convert stream T to Collection<T> to stream T or Collection<A> to Collection<B> via some function...

If you added more generics you could actually do some wild stuff, I'm talking monadic endofunctors and categorical enrichment, or just more nice general datastructures like Map. 

# What We Want

We're building a more generally composible distributed system on top of protobuf and grpc, where implementing a new service that takes four others and just does some basic Reduce<Accumulation>(Map<Transformation>(Foo(Bar(Baz(msg, n).data)))) > Collect<T>("data.db") (but maybe a different language syntax/structure) kind of stuff to create a new API. Basically everything getting in the way of this is boilerplate with no essential complexity.

To do this it's very important that we can search, access, and resolve types efficiently. So we are building a distributed, namespaced type system to do that. But proto makes that kind of annoying because it groups many message proto definitions into single files and doesn't really optimize for information-density. Ideally, each message/service/other important type could be resolved as an individual file, and have a 1:1 mapping between its namespace/package/name and URL; then we don't even need the wrapping message anymore and can just enumerate the fields. Also, we don't want to deal with field numbers because making them positional and dense at the cost of not being able to rename fields at the interface level is a much better tradeoff than the alternative.

We also need to make it easy to integrate with relational/tabular queries, stream processing, "scripted" or other various cool and exciting things without defining wrappers and FooRequest and FooReponse all the time. You already know that the instant we have Collection<T>, we shall never speak of "repeated" again.

oneof needs to go live in a farm upstate and get replaced by the much cooler union type, and we'll introduce product types/tuples and allow them to be treated as fields bundling multiple types of data. But we'll call them a coproduct type and product type because we're going to embrace category theory.

We're also generally going to make it easier to define inline types and eliminate boilerplate while adding some syntactic sugar like Foo[5] foo; and Bar b1, b2;

Now you might ask "but what about forward compatibility?" because it sounds too good to true. But no, you never needed field numbers if you were willing to leave the damn schema alone and keep their fields in order while they still exist, and replace them with:

foobardemo.protostar:

```protostar
Foo&Foo&Foo foo_triplet // product type
reserved // don't be weird about it, just leave it there and change its name to reserved
Bar&reserved&Foo bartup // you can deprecate fields within product types like so 
reserved[5] // it's that easy
Baz|reserved baz // coproduct type, deprecated field, still need to leave it at the end, but can add subsequent fields if necessary
Collection<Foo&Bar> foobars // looks like a table doesn't it. 
Status StatusOk | StatusFailed | StatusPaused | reserved | StatusReserved // much more likely to use enums now that they don't feel gross
Foo&(Bar|(Foo&Baz)) fooand // feels a little too much like code golf but maybe you want to do this sometimes
Any stuff // same old any
Duration|Timestamp tdata // same 
Status cur,prev // counts as two lines basically. Note we are reusing a previously-defined enum Status.
int|Empty maybe_int // same old empty
Option<int> optional_int // empty in disguise
```

foobardemo.foo.protostar:

```protostar
types.accretional.net/somebody/something/thing.protostar thing // use their thing
Foobardemo demo // parent-child corecursion probably ok (TODO: validate).
Status internal_status // defined in parent
```

foobardemo.bar.protostar:

```protostar
string|int // aliasing, be careful because changing this is always breaking
```

foobardemo.baz.protostar:

```
types.accretional.net/somebody/something/thing.protostar // generally you probably want aliasing more like this
```

foobardemo.imports.protostar:

```
types.accretional.net/somebody/something/thing.protostar
```

foobardemo.pkg.protostar:

```
go github.com/example/foo/v1;foov1
```

vendored/types.accretional.net/somebody/something/v1/thing.protostar:

```
string id // and more stuff. Note the v1 above
```

Ok well what if you accumulate a bunch of tech debt and the reserved keyword everywhere starts bothering you? By default major version bumps cleanup all reserved keywords, and old and new versions know which fields to switch between then by saving the diff and making it accessible.

## Generics

Here's how Collection<T> works:

collection-1.protostar:

```
A type_name
bytes data // custom parsing
```

Here's Map<K, V>:

map-2.protostar:

```
Collection<A&B>
```

Here's Optional<T>:

optional-1.protostar:

```
Empty | A
```

Here's Tree<T>:

tree-1.protostar:

```
A node
Optional<Collection<A>> children
```

## Services

service.protostar.api:

```
methodname: input, output
pickone: Foo&Bar, Foo|Bar
reqresp: (Bar bar, Baz[2] baz, Optional<int> length), (string name, int len) // define request and response types implicitly
pack-1 Any, A // generic on method, Pack<T>
listfoo: Collection<string>, Collection<Foo> // worth it
```

Note, we'll make this a bit more complex later by introducing some build configs for the service's package init/grpc service/state, something like

service.protostar.api.init:

```
initmethod(string, "yo") // or something like this? where initmethod is like initmethod: string&string, Empty or something
```

service.protostar.api.interceptors:

```
myauth
mylogging
```

service.protostar.api.state:

```
Collection<Log> logs // might be accessible via init or checked for by methods for integration?
```

service.protostar.api.lifecycle:

```
coldstart_max("500ms")
sigterm_max("5s")
idletime_max("1m")
```
