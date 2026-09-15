# openapi-nv

OpenAPI is a description format for HTTP APIs: one document lists every
path, the operations on it, what each accepts and what each answers.
The current version is
[OpenAPI Specification 3.1.1](https://spec.openapis.org/oas/v3.1.1.html),
whose schema objects are
[JSON Schema 2020-12](https://json-schema.org/draft/2020-12). This
package brings that document to novo-lang as a typed value: built by
hand or derived from a route table, rendered to and read from JSON,
compiled into a request validator, and compared with an earlier version
to say what breaks.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What an OpenAPI document is

The document's **paths object** maps a **path template**, such as
`/users/{id}`, to a **path item**. A path item holds one **operation**
per HTTP method. An operation has an `operationId`, a list of
**parameters**, an optional **request body** and a set of
**responses** keyed by status code.

A **parameter** has a name and a location: `path`, `query`, `header` or
`cookie` (section 4.8.11). It also has a **style** and an **explode**
flag, which together say how a value with structure is written into a
string. Section 4.8.11.5 fixes a default style per location.

A schema is written as a JSON Schema document. It may be written
**inline**, or as a **`$ref`**: a JSON Pointer such as
`#/components/schemas/Pet` into this document, or a URI naming another
one. The **components object** is where a document keeps schemas,
responses and security schemes for `$ref` to point at.

A **security requirement** names a **security scheme** the document
declares, such as a bearer token or an API key. The document has a
default list and an operation may override it.

A **vendor extension** is a member whose name begins `x-`. The
specification allows one anywhere, and gateways, generators and
renderers each carry what they need in them.

This package also takes a **route table** from
[router-nv](https://novo-lang.org/packages/router-nv): the patterns a
service actually dispatches on. A paths object and a route table are
the same facts written twice, and this package can derive the first
from the second or compare the two.

Every function in this package performs no input and no output. It
resolves no `$ref` that names another document, opens no socket, and
never calls an `openIdConnect` discovery URL.

## Install

```
novo pkg add openapi-nv
```

## Example

```novo
use std.json
use std.list
use oapidoc
use oapijson

fn main() [io]
    // What the operation answers: a 200 whose body is a JSON object.
    let ok = oapidoc.with_content(
                 oapidoc.response("200", "the user"),
                 "application/json",
                 OapiSchemaInline(json.object([("type", json.string("object"))])))

    // One operation, on one path, in one document.
    let doc = oapidoc.with_path(
                  oapidoc.document("Users", "1.0.0"),
                  oapidoc.path_item("/users/{id}", oapidoc.operation(OapiGet, "getUser", ok)))

    // Everything wrong with the document itself. An empty list means it
    // can be compiled into a validator.
    println("${list.len(oapidoc.check(doc))} problems with the document")

    // The document as JSON text, which is what a client generator reads.
    println(oapijson.to_json_text(doc))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: openapi-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `oapidoc` | The document as a value, the builders over it, the reading rules the specification defines, and the checks on the document itself. |
| `oapijson` | The document to and from the standard library's JSON value, and the version questions asked before reading one. |
| `oapiroute` | The paths object derived from a route table, the comparison between the two, and the route table derived back out. |
| `oapivalid` | A document compiled once, and a request or a response checked against it. |
| `oapidiff` | What changed between two documents, and whether each change breaks a client or a server. |

## How to choose an entry point

**`oapidoc.document` and its `with_*` functions build a document by
hand.** Use them when the description carries prose a route table
cannot know.

**`oapiroute.from_routes` derives a document from a route table.** Use
it when the description should follow the service without anyone
maintaining it. **`oapiroute.drift` compares the two** and answers what
differs, which is what a hand-written document plus a CI gate wants.

**`oapivalid.compile` is called once, at start-up**, and
`validate_request` many times. `validate_request_parts` takes the query
string and the body separately, for a caller that has them apart.

**`oapidiff.diff` compares two documents** and `is_breaking` answers,
per audience, whether a change breaks anyone. `is_compatible` and
`suggested_bump` are the two one-call forms.

## The rules a user needs

1. **`validate_request` cannot fail.** It answers a list of failures. A
   request that does not match its operation is the answer, and it is a
   400 whose body is that list. Everything that can genuinely be wrong
   is wrong in the document, and `oapivalid.compile` is where that is
   reported.
2. **Compile once and validate many.** `compile` resolves every `$ref`
   and hands every schema to schema-nv once. A validator that walked the
   document per request would recompile the same schemas on every call.
3. **A failure names three places**: where in the request, as a
   parameter, a header or a JSON Pointer into the body; where in the
   document the rule came from, as a JSON Pointer; and what was
   expected. The document pointer is what turns a rejection into
   something an API's author can act on.
4. **A path item's parameters merge into its operations'.** An
   operation's entry overrides a path item's of the same name and
   location (section 4.8.9). A caller reading `op.parameters` alone
   misses half the parameters on a well-written document, so
   `oapidoc.parameters_of` takes both and is a function rather than a
   field.
5. **An empty `security` on an operation means public.** An operation's
   own list replaces the document's, and an empty own list makes the
   operation unauthenticated on a document that is otherwise
   authenticated (section 4.8.2). `OapiOperation.has_own_security` is
   what distinguishes an empty override from an absent one.
6. **`oapivalid.security_present` checks presence, never validity.**
   Validating a bearer token needs a key, and this package holds none.
   A service that took presence for authentication would have a check
   any client passes by sending a header.
7. **A body arrives as a JSON value the caller parsed.** A request
   whose body is not JSON at all is the caller's own 415 and never
   reaches here.
8. **`validate_response` belongs in a test suite and in staging.** A
   service must not answer 500 because its own description is stale.
9. **OpenAPI 3.0 is refused by name, not partly read.** A 3.0 schema
   object is JSON Schema draft-04 with OpenAPI's own extensions:
   `nullable: true` rather than `type: ["string", "null"]`, and
   `exclusiveMinimum` as a boolean rather than a number. Reading one as
   3.1 gives wrong answers about what a field accepts rather than no
   answer. `oapijson.declared_version` and `supported_versions` let a
   caller say something better than "unsupported".
10. **A wildcard route has no OpenAPI spelling.** 3.1 has no wildcard
    path templating, and the nearest approximation, a path parameter
    whose value contains slashes, is rejected by some tools and read
    differently by others. `oapiroute.path_template` answers nothing
    for such a pattern and `is_describable` is the question on its own.
11. **A route table cannot know what an operation is called or what it
    answers.** `OapiRouteMeta` is where the caller supplies those, keyed
    by the route identifier the service already dispatches on, so the
    description lives beside the handler.
12. **Breaking is asymmetric, so `is_breaking` takes an audience.** The
    same change can be safe for a client and breaking for a server.

    | Change | Breaks clients | Breaks servers |
    | --- | --- | --- |
    | A new required request field | yes | no |
    | A new optional request field | no | no |
    | A request field made optional | no | no |
    | A new response field | no | yes |
    | A response field removed | yes | no |
    | An enum value added to a request | no | yes |
    | An enum value added to a response | yes | no |
    | A changed `operationId` | yes | no |
    | An added security requirement | yes | no |

    A changed `operationId` breaks clients because generated clients
    name their methods after it.
13. **A changed schema is reported as breaking.** Two structurally
    different schemas can describe the same values, and deciding that
    is subtyping, which is undecidable in general for JSON Schema. The
    change carries both schemas, `is_breaking` answers `true`, and a
    person overrules it.
14. **A `$ref` to another document is named, never followed.**
    `oapidoc.external_refs` answers the URIs. The caller reads them
    however it likes, registers each with schema-nv, and hands the
    registry to `oapivalid.compile`. A document from outside therefore
    cannot make a program fetch a URL.
15. **Schemas, examples and `x-` extensions stay as raw JSON.** Each is
    arbitrary by definition, and the extensions survive in the order
    they arrived, because a parser that dropped them would round-trip a
    document to something its owner does not recognise.
16. **Read YAML with a YAML parser.** See "What is not included".

## Writing schemas

The schemas in a novo-lang OpenAPI document are written by hand, as
JSON Schema values. `OapiSchemaRef` carries a JSON value for that
reason.

The language has no compile-time view of a declaration's fields.
`@derive(...)` is reserved and refused (SPEC section 1234). What exists
instead is implicit structural implementations (SPEC section 3.8.1):
every struct gets `Serialize` and `Deserialize` wherever its fields do,
synthesised on demand. That is a run-time walk over a value through a
`Serializer`, so a library can observe the shape of one value and never
the shape of a type.

| What a declaration says | What a walk of one value gives |
| --- | --- |
| Every field's name and type | The names and kinds that value happened to carry |
| An optional field is not required | Nothing; an absent field and a present empty one look alike |
| Every variant of an enum | The one variant that value was |
| A doc comment becomes a description | Nothing; comments are not in a value |
| A declared minimum becomes a constraint | Nothing |
| A field never constructed in this program | Nothing |

## What is not included

- **YAML.** A YAML document and a JSON document share one data model,
  and the specification allows either. Parse with
  [yaml-nv](https://novo-lang.org/packages/yaml-nv) and hand the value
  to `oapijson.from_json`. A `from_yaml` here would make every JSON
  caller pay for a YAML parser.
- **OpenAPI 3.0.** See rule 9. A converter is its own tool.
- **Code generation.** A generator reads this document and is a
  different package.
- **A documentation renderer.** HTML from a document is
  [html-nv](https://novo-lang.org/packages/html-nv) and
  [tera-nv](https://novo-lang.org/packages/tera-nv)'s work.
- **Swagger 2.0, AsyncAPI and gRPC reflection.** Different documents
  with different models.
- **Fetching a `$ref`.** See rule 14.
- **Content negotiation.** `oapivalid.content_type_for` matches a media
  type against an operation's, which is the subset a document needs.
  [mime-nv](https://novo-lang.org/packages/mime-nv) owns the full rules.
- **A build for a microcontroller.** A document is a tree of growable
  lists and the JSON value it renders to is a host handle, so this
  package makes no device claim.

## Related packages

- [router-nv](https://novo-lang.org/packages/router-nv) is the route
  table a document can be derived from or checked against. This package
  depends on it.
- [schema-nv](https://novo-lang.org/packages/schema-nv) compiles and
  applies JSON Schema, and its validation cannot fail either. This
  package depends on it.
- [graphql-nv](https://novo-lang.org/packages/graphql-nv) describes the
  other API style. Take it when the client chooses its fields. Take
  this one when the client asks for a fixed resource.
- [yaml-nv](https://novo-lang.org/packages/yaml-nv) parses the format
  most OpenAPI documents are written in.
- [mime-nv](https://novo-lang.org/packages/mime-nv) is content
  negotiation done properly.
- `std.json` in the standard library is the value a document renders to
  and is read from.

## Tests

```bash
novo test tests/oapidoc_tests.nv     # the document, the merge rules, the JSON round trip
novo test tests/oapiroute_tests.nv   # the derivation, drift, validation and the diff
```

The normative source is the OpenAPI Specification 3.1.1, with JSON
Schema 2020-12 through schema-nv. The reference implementations are
utoipa, for the shape of a derived document, FastAPI, for validation
answering every problem rather than the first, and openapi-diff, whose
breaking-change asymmetry is the table in rule 12. The vectors are the
specification's own examples, the openapi-diff compatibility suite, and
FastAPI's generated documents for the parameter serialisation cases.

The suite asserts the cases where two well-known tools disagree about
what a document says: a path item's parameters merging into an
operation's, an empty `security` on an operation meaning public rather
than absent, a 3.0 document refused rather than read as 3.1, and the
`x-` members surviving a round trip in order.

The tests compile today and fail at run, each on the
`not implemented: openapi-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `pub struct` and `pub enum` in the five modules | the types are declared |
| `oapidoc.document`, `.with_server`, `.with_path`, `.with_component_schema`, `.with_tag` | no |
| `oapidoc.path_item`, `.with_operation`, `.operation`, `.response`, `.with_content`, `.parameter` | no |
| `oapidoc.find_operation`, `.operations`, `.parameters_of`, `.security_of`, `.template_params` | no |
| `oapidoc.external_refs`, `.dangling_refs`, `.resolve_local`, `.check` | no |
| `oapidoc.method_name`, `.method_named`, `.param_in_name`, `.default_style`, `.default_explode` | no |
| `oapidoc.OapiError.message` | no |
| `oapijson.to_json`, `.to_json_text`, `.to_json_pretty` | no |
| `oapijson.from_json`, `.from_json_text`, `.from_json_lenient` | no |
| `oapijson.looks_like_document`, `.declared_version`, `.supported_versions` | no |
| `oapijson.schema_ref_json`, `.schema_ref_of` | no |
| `oapiroute.path_template`, `.route_pattern`, `.is_describable` | no |
| `oapiroute.from_routes`, `.path_items`, `.drift`, `.to_routes` | no |
| `oapiroute.empty_meta`, `.find_meta` | no |
| `oapivalid.compile`, `.validate_request`, `.validate_request_parts`, `.validate_response` | no |
| `oapivalid.operation_pointer`, `.security_present`, `.content_type_for` | no |
| `oapivalid.failures_in`, `.failures_json`, `.failure_text`, `.failure_in_name` | no |
| `oapivalid.decode_param`, `.encode_param` | no |
| `oapidiff.diff`, `.is_breaking`, `.breaking`, `.is_compatible` | no |
| `oapidiff.change_text`, `.change_pointer`, `.changes_json`, `.suggested_bump`, `.schema_eq` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
