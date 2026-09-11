# openapi-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

OpenAPI 3.1 as a value: a document you read as fields rather than walk
as JSON, a paths object **derived** from the route table instead of
typed beside it, requests validated against their operations as pure
arithmetic, and a diff that says which changes break a client.

- `oapidoc` — the typed document, and `$ref` as a sum;
- `oapijson` — to and from the standard library's JSON value;
- `oapiroute` — the paths object from a router-nv table, and `drift`;
- `oapivalid` — compile once, validate many, and never fail;
- `oapidiff` — what changed, and whether it breaks somebody.

```
novo pkg add openapi-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use oapiroute
use routetree

// The question a CI job asks: does this document still describe this
// service?  An empty answer is the whole point.
fn describes(doc: oapidoc.OapiDoc, t: routetree.RouteTree) -> Bool
    list.len(oapiroute.drift(doc, t)) == 0
```

## The load-bearing interface: `validate_request` cannot fail

```novo ignore
pub fn validate_request(c: OapiCompiled, method: Str, path: Str,
                        headers: [(Str, Str)], body: JsonValueH) -> [OapiFailure] []
```

It answers a **list**, never a `Result`. A request that does not match
its operation is **the answer** — it is a 400 whose body is that list —
and modelling it as an error would make the ordinary case travel the
failure path.

Everything that can genuinely be wrong is wrong in the **document**, and
`compile` is where that is said: an unresolvable `$ref`, a schema
schema-nv refuses, a path template whose parameters no operation
declares. schema-nv makes the same split for the same reason, and the
two halves compose because of it.

**Compile once, validate many.** `compile` resolves every `$ref` and
hands every schema to `schcompile.compile_with`, once, at start-up. A
validator that walked the document per request would parse the same
schemas on every call — which is the shape that makes people turn
validation off in production.

**Every failure names three places**: where in the request (which
parameter, which header, which JSON Pointer into the body), where in the
**document** the rule came from, and what was expected. All of them
rather than the first, because a form with six bad fields should tell a
person about six and an agent should be told everything wrong with its
tool call.

The document pointer is the field that turns a rejection into something
an API's author can act on. *"Your request is wrong"* is a complaint;
*"`/paths/~1users/post/requestBody/.../age/minimum` says 0"* is a
conversation.

## Derived, not typed twice

A route table and a paths object are the same facts written twice, and
the copy that is not executed is the one that goes stale. A service adds
an endpoint, nobody updates the description, and six months later the
description is a document about a service that no longer exists.

`oapiroute.from_routes` derives the second from the first.
`oapiroute.drift` checks them against each other, which is what a
service with prose in its description wants — it keeps the hand-written
document and gates on the comparison.

**A catch-all has no OpenAPI spelling, and `path_template` refuses
rather than approximating.** OpenAPI 3.1 has no wildcard path
templating. The nearest approximation — a path parameter whose value
contains slashes — is rejected by most tools and read differently by the
rest, so emitting one would produce a document that generates clients
which do not work. A service serving `/files/*rest` documents that
endpoint by hand or not at all, and a named refusal at build time is a
better way to find out than a generated client.

What a route table knows: the path, the parameters, the methods. What it
cannot know: what an operation returns, what it accepts, what it is
called, what it is for. `OapiRouteMeta` is where the caller supplies
those, **keyed by the route id it already dispatches on** — so the meta
lives beside the handler rather than in a second table that can fall out
of step with the first.

## What utoipa derives at compile time, and where the line falls here

utoipa's `#[derive(ToSchema)]` reads a Rust struct's fields **at compile
time** and emits a JSON Schema; `#[utoipa::path(...)]` on a handler
emits an operation. The document is generated from the types the service
already has, and it cannot drift from them because it *is* them.

novo-lang cannot do that today, and the reason is precise.
`@derive(...)` is reserved and refused (SPEC § 1234); what the language
has instead is **implicit structural impls** (§ 3.8.1): every struct
gets `Serialize` and `Deserialize` wherever its fields do, synthesised
on demand. That is a **run-time walk through a `Serializer`**, not a
compile-time view of the declaration — so a library can observe the
shape of a *value* and never the shape of a *type*.

The gap that leaves, named:

| utoipa reads from the declaration | what a `Serializer` walk of one value gets |
| --- | --- |
| every field's name and type | the names and kinds **the sample took** |
| `Option<T>` → not required | nothing — a present `None` and an absent field look alike |
| every enum variant | the one variant the sample was |
| doc comments → `description` | nothing; comments are not in the value |
| `#[schema(minimum = 0)]` → constraints | nothing |
| a field never constructed in this program | nothing |

So the schemas in a novo-lang OpenAPI document are **written by hand**
today, as JSON Schema values, and that is why `OapiSchemaRef` carries a
`JsonValueH` rather than a type built from a struct.

The half that *would* close with no language change is a `Serializer`
implementation that records a shape instead of writing bytes — good
enough for field names and kinds from a sample value, and honest about
the four rows above that it cannot fill. It is not in this interface
because a schema that is right about names and silently wrong about
optionality is worse than one a person wrote. What would close the whole
gap is compile-time access to a declaration's fields, which is a
language question and not this package's.

## The diff compares meaning, and breaking is asymmetric

`git diff` on two OpenAPI files reports a reordered `paths` object as a
hundred changed lines and a `required: true` added to a request body as
one — and the second is the one that breaks every client.

A change is breaking for a **client** when the service will now refuse
something it used to accept, or stop sending something a client relied
on receiving. The asymmetry runs the opposite way on the two sides, and
a tool that got it backwards would bless exactly the changes that cause
an outage:

| change | breaks clients | breaks servers |
| --- | --- | --- |
| a new **required** request field | yes | no |
| a new **optional** request field | no | no |
| a request field made optional | no | no |
| a new response field | no | yes |
| a response field **removed** | yes | no |
| an enum value added to a **request** | no | yes |
| an enum value added to a **response** | yes | no |
| a changed `operationId` | yes — generated clients name methods after it | no |
| an added security requirement | yes | no |

So `is_breaking` takes an `OapiAudience`, and a change that is safe one
way and breaking the other is two answers from one function rather than
a caller's own judgement.

**What a diff cannot see**: a schema's *meaning*. Two structurally
different schemas can describe the same values — `type: ["string",
"null"]` and an `anyOf` of the two — and deciding that is subtyping,
which is undecidable in general for JSON Schema. So a schema change is
reported with both schemas carried and `is_breaking` answers `true`:
the safe direction for a tool that gates a release, and a human
overrules it.

## The layer, and the `$ref` that is not followed

`core` — no effects. A document is a value, rendering it is a walk, and
validating a request against it is arithmetic over values the caller
already holds.

The one place a document reaches outside itself is a `$ref` to another
document, and resolving one means **reading**:

```novo ignore
let wanted = oapidoc.external_refs(doc)     // core: "these URIs"
// ... the host reads each one, however it likes ...
let reg = schcompile.with_document(reg, uri, fetched)
let compiled = oapivalid.compile(doc, reg)
```

That is `docs/publishing.md` § How a `core` package takes bytes from its
host, in its third shape — the core asks, the host performs — and it is
schema-nv's own shape, which is why the two compose rather than each
having an opinion about fetching. It is also a security property: **an
OpenAPI document from outside cannot make a program fetch a URL**,
because the program decides what it fetches and this package has no
`[net]` to do it with. `openIdConnect`'s discovery URL is never called
for the same reason.

**No `@tier(embedded)` claim, and none is intended.** A document is a
tree of growable lists, and the JSON value it renders to is a host
handle — `json.*` is not admitted at `@tier(embedded)`. Nothing on a
device reads an API description. The audit's `core-embedded` row passes
as *makes no device claim*.

## YAML is yaml-nv's job

Most OpenAPI documents in the world are YAML, and a reader will look for
`from_yaml` first. It is not here, and the reason is that YAML and JSON
are the same data model:

```novo ignore
let value = yaml.parse(text)            // yaml-nv's job
let doc   = oapijson.from_json(value)   // this package's
```

A `from_yaml` here would be a dependency on a YAML parser that every
JSON caller would pay for. The specification agrees: it says a document
may be written in either format and defines its semantics over the
shared model.

**3.0 is refused rather than partly read.** A 3.0 document's schema
object is JSON Schema draft-04 with OpenAPI's own extensions —
`nullable: true` rather than `type: ["string", "null"]`,
`exclusiveMinimum` as a boolean rather than a number — so reading one as
3.1 gives *wrong* answers about what a field accepts rather than no
answer. `declared_version` lets a caller say something better than
"unsupported"; a converter is a real tool and it is not this one.

## The three places a document stays raw JSON

A schema, an example, and a vendor extension. Each is by definition
arbitrary, and a type over them would be a type that refuses valid
documents.

The `x-` extensions matter most: every gateway, code generator and
documentation renderer carries what it needs in them, and a parser that
dropped them would round-trip a document to something its owner does not
recognise. They survive, in the order they arrived.

## Two rules that OpenAPI tooling gets wrong, made into functions

**A path item's parameters merge into its operations'**, with an
operation's entry overriding a path item's of the same name and
location. A caller reading `op.parameters` alone misses half the
parameters on a well-written document — the single most common bug in
OpenAPI tooling — so `parameters_of` is a function rather than a field.

**An empty `security` on an operation means public.** An operation's own
list *replaces* the document's, and an empty own list makes the
operation unauthenticated on a document that is otherwise authenticated.
Without `has_own_security` an empty override and an absent one are the
same value, and the difference is whether an endpoint needs a token.

## What validation does not do, and why each is somebody else's

- **Authenticate.** `security_present` checks a security requirement for
  *presence* and never validity. Validating a bearer token means knowing
  a key, and this package has none and should not — a service that took
  presence for authentication would have a check any client passes by
  sending a header.
- **Decode a body.** `body` arrives as a `JsonValueH` the caller parsed,
  so a request whose body is not JSON at all is the caller's own 415 and
  never reaches here.
- **Negotiate.** Matching `Content-Type` against an operation's media
  types is essence-and-wildcard comparison. mime-nv owns that properly;
  `content_type_for` is the subset a document needs, and mime-nv is the
  dependency this package would take if the two ever have to agree about
  a `+json` suffix.
- **Run in production as a response validator.** `validate_response` is
  right in a test suite and in staging. A service must not 500 because
  its own description is stale.

## Where the names come from

Public type names are unique across the whole assembly, dependencies
included.

| here | the obvious name | why not |
| --- | --- | --- |
| `OapiDoc` | `Document`, `OpenApi` | `TomlDoc`, `YamlDoc`, `JsonDoc` and `XmlDoc` are the standard library's precedent for exactly this |
| `OapiOperation` | `Operation` | generic enough to collide with anything |
| `OapiParam` | `Param`, `Parameter` | `HttpParam` is the standard library's |
| `OapiSchemaRef` | `SchemaRef`, `Ref` | `Ref` is url-nv's, and schema-nv owns the `Sch` prefix |
| `OapiFailure` | `Failure`, `Error` | `SchFault` is schema-nv's neighbour and `Error` is a prelude trait |
| `OapiChange` | `Change`, `Diff` | generic |
| `OapiMethod` | `Method` | http-codec-nv publishes `H1Method` and router-nv `RouteMethod`; a bare `Method` would collide with both |
| module `oapidoc`, `oapivalid`, … | `openapi`, `doc`, `validate`, `diff` | the last three are names other packages will want, and `openapi` would collide with a future client generator's own module |

Prefixed `Oapi` rather than `OpenApi` because every variant carries it
too — `OapiInPath` rather than `InPath` — and a variant is constructed
by name across the whole assembly.

## The reference implementation

**utoipa** for the shape of a derived document and for the
compile-time line drawn above. **FastAPI** for the observation that a
description derived from the code is the only kind that stays true, and
for validation answering *every* problem rather than the first.
**openapi-diff** for the breaking-change table, whose asymmetry is the
part worth porting. **The OpenAPI Specification 3.1.1** for everything
normative, and **JSON Schema 2020-12** through schema-nv, because 3.1's
headline change is that its schema object *is* JSON Schema.

The oracles are the specification's own examples, the `openapi-diff`
compatibility suite, and FastAPI's generated documents for the parameter
serialisation cases.

Deliberately left out, and where it goes instead:

- **YAML.** yaml-nv, in front of `from_json`.
- **OpenAPI 3.0.** Refused by name; a converter is its own tool.
- **Code generation.** A generator reads this document and is a
  different package — and a good one needs the compile-time line above
  to move first.
- **A documentation renderer.** HTML out of a document is html-nv and
  tera-nv's business.
- **Swagger 2.0 / AsyncAPI / gRPC reflection.** Different documents
  with different models.
- **Fetching a `$ref`.** `core`. `external_refs` is the seam.

## Status

Every function is `todo()`. Two suites, both red, both for the same
reason — every assertion reaches `not implemented: openapi-nv.<fn>`,
which is the expected result until the bodies land.

```
novo test --isolate tests/oapidoc_tests.nv     # the document, the merge rules, the JSON round trip
novo test --isolate tests/oapiroute_tests.nv   # the derivation, drift, validation and the diff
```

`novo doc` renders and its examples compile.
