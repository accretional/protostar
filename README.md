# protostar
Enhanced protobuf builds, testing, and codegen

Warning - messy draft/design phase

Make it easy to expand from fully self-contained .protostar protobuf/grpc/language implementation definitions all the way out to e2e tests, client libraries, example service implementations, and grpc method impl stubs.

# Format

Markdown-ish with interpspersed code blocks of proto code blocks (interpreted as belonging to the .protostar by default), various impl language code blocks (matched to proto code), actualy markdown contents for documentation, and specially:

protostar codeblocks (interpreted as being direct protostar configs), structured with code block prefixes containing eg

* protostar-proto - formatting, splitting, creation of constituent .proto files
* protostar-lang - impl language configuration and settings, client code gen
* protostar-tests - fuzz, validation, e2e test config/setup
* protostar-LANG - language-specific soure code
* protostar-unit-LANG  language-specific unit test soure code
* protostar-inits - package level inits
* protostar-build - booter/loader/main/etc info, Dockerfile info, bufgen/api gateway config stuff
* protostar-clients - specific client configuration, set client interceptors, etc
* protostar-services - specific service configuration, set service interceptros, etc.

TODO: think about protostar-mdsvex, protostar-statue, protostar-collector, etc.

Very rough WIP. NOTE: hold off on main, grpc servie testing unitl unit testing and basic packaging / splitting working first.

# Concept

For each a directory eg foo/ with a .protostar, each representing a kind of intermediate .proto file within a shared foo/ package

Build each .protostar separately/independently, save eg each message/service from foo/bar.protostar to foo/proto/bar/msg/messagename.proto

```
foo/proto/bar/service/servicename.proto
foo/proto/baz/msg/othermessagename.proto
foo/proto/baz/service/bazservice.proto
foo/proto/baz/service/bazcollection.proto
```

Protostar automatically generates service impl wrappers

```
foo/service-gen/bar/go/ServiceName.go
foo/service-gen/bar/py/ServiceName.py
```

These can be paired with automatically generated tests using fuzzed values (enabled by default), specific input-output pairs, or e2e service tests.

```
foo/test/bar/service/servicename-fuzz.textpb
foo/test/bar/service/servicename-pairs.textpb
foo/test/bar/service/servicename_test.go 
foo/test/bar/service/servicename.go
foo/test/bar/service/validationset.textproto
```

We also get method impl stubs by default, a path to another file or function implementing the logic, or a defined impl from the .protostar

```
foo/method-gen/bar/servicename/DoMyThing.go
foo/method-gen/bar/servicename/DoMyThing.py
```

Each protostar can set 1+ package level inits with logic similar to methods

```
foo/init-gen/bar/MyInit.go
foo/init-gen/bar/MyInit.py
```

Each package ends up witha single protostar package/content wrapper for each language 

```
foo/go/Protostar.go
foo/py/protostar.py
```

Also potentially main for each language

```
foo/go/main.go
foo/py/main.py
```

Also potentially a booter (only runs on main startup, not on package startup), loader (start and load dynamically via a lib), or lib (non-proto code) or Dockerfile, etc.

```
foo/go/Protostar.go // may contain booter
foo/go/lib/Code.go
foo/go/Dockerfile
foo/py/protostar_loader.py
foo/py/lib/code.py
foo/py/Dockerfile
```

Any language-specific unit tests also separated by language

```
foo/go/protostar_test.go
foo/py/protostar_test.py
```

Then if you want full customization of program execution/integration you can just import in eg example/go/Protostar_example.go and let er rip.
