rubberbuf
=========

rubberbuf is a clojure(script) library to parse protobuf definition (.proto) into abstract syntax tree (AST).

It can be used to develop tools or libraries that transpile protobuf definitions into other langs like C/C++, Java, Python, etc, or to dynamically read/analyze protobuf definitions.

## Usage
Add the following to deps.edn (or its equivalent for lein).
```edn
{:deps
 s-expresso/rubberbuf {:git/url "https://github.com/s-expresso/rubberbuf.git"
                       :git/sha "371021dafbc3065aa1b4e8c3396b02d064cba1b5"
                       :git/tag "v0.1.7"}}
```
then in code
```clojure
(ns my-ns
  (:require [rubberbuf.core :refer [protoc]]))

; optionally add tap to be notified of compilation progress
;(add-tap println)

; :auto-import and :normalize both default to true if omitted
(protoc ["/path/to/protobuf1/", "/path/to/protobuf2/"] ; paths
        ["file1.proto" "file2.proto"]                  ; files
        ; :auto-import true, :normalize true
        ) 
```
produces a registry of AST, i.e. map of file name to AST
```edn
{"file1.proto" [[:syntax "proto3"]
                [:package "a.b.c"]
                [:import "file3.proto"] ...]
 "file2.proto" [[:syntax "proto2"]
                [:package "x.y.z"] ...]
 "file3.proto" [...]}
```
Note
* `:auto-import true` instructs `protoc` to recursively load all imports
* hence `file3.proto` is in registry because `file1.proto` imported it
* if `file3.proto` imports another file, it will also appear in the registry
* circular import is ok

## AST Format
rubberbuf's AST format follows the original protocol buffer definition closely and only adds some keywords or substitute some protobuf syntaxes with keywords. Interpretation of the AST should be simple and easy.

Some less obvious features are shown using example below.
```protobuf
// content of example.proto
syntax = 'proto2';
package my.package.ns;

enum Enm1 {
  option allow_alias = true;
  ZERO = 0;
  ONE = 1;
  ANOTHER_ONE = 1 [deprecated = true];
}

message MsgA {
  optional Enm1 field_a1 = 1;
  optional sint32 field_a2 = 2 [deprecated = true, default = 5];
  message MsgB {
    optional uint32 field_b1 = 1;
    extensions 1, 2, 1000 to 2000;
  }
  repeated MsgB field_a3 = 3;
  extend MsgB {
    optional bool ext_1 = 1000;
  }
}
```

will yield
```edn
{"example.proto"
 [[:syntax "proto2"]
  [:package "my.package.ns"]
  [:enum "Enm1"
    [:option "allow_alias" :true]
    [:enumField "ZERO" 0 nil]
    [:enumField "ONE" 1 nil]
    [:enumField "ANOTHER_ONE" 1 [["deprecated" :true]]]]
  [:message "MsgA"
    [:field :optional "my.package.ns/Enm1" "field_a1" 1 nil]
    [:field :optional :sint32 "field_a2" 2 [["deprecated" :true]
                                            ["default" 5]]]
    [:message "MsgB"
      [:field :optional :uint32 "field_b1" 1 nil]
      [:extensions 1 2 [1000 2000]]
      [:field+ :optional :bool "my.package.ns/MsgA.ext_1" 1000 nil]]
    [:field :repeated "my.package.ns/MsgA.MsgB" "field_a3" 3 nil]]]}
```
With the following note worthy characteristics:
* `[:field ...]` and `[:enumField ...]` has a `nil` at the end if it has no field option
* `[:field ...]` of primitive type uses a keyword like `:sint32`
* `[:field ...]` of message/enum type uses string like `"MsgA"` 

And the following are due to `:normalize true`
* `[:field ...]` of message/enum type resolved to a fully qualified name like `"my.package.ns/Enm1"`
* `[:field+ ...]` is injected into message from field inside `extend` 
* `[:field+ ...]` name `"my.package.ns/MsgA.ext_1"` follows google's naming convention; long and verbose but an necessary evil to avoid name collision
* `extend` is removed from AST since already injected

## AST Transformation
`rubberbuf.ast-postprocessing` provides transformation function that can be applied to above output.
* unnest: nested message/enum are extracted out to top level, with its name replaced with a scoped name (.e.g `MsgA.MsgB.MsgC`)

## Unsupported feature
* protobuf `group` type (deprecated by google)

## Known Issues
* no checking of semantic error (yet), i.e. no checking that same field number/name is reused in the same message, etc.
* some types of protobuf syntax error can trick parser into looping indefinitely until out of memory

Above issues imply rubberbuf is not ready to be used as a standalone transpiler, but the issues can be worked around by using google's protoc (protocol buffer compiler) to validate the protobuf files before feeding into rubberbuf. This isn't ideal, but is a reasonable workaround especially if the need to invoke googles' protoc compiler already pre-exists.
