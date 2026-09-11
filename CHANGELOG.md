# Changelog

All notable changes to openapi-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `oapidoc` — OpenAPI 3.1 as a typed value, with `$ref` as a sum and
  `external_refs` as the layer seam; `parameters_of` and `security_of`
  carry the two merge rules tooling gets wrong.
- `oapijson` — to and from the standard library's JSON value, strictly,
  with the `x-` extensions kept.
- `oapiroute` — `from_routes` derives the paths object from a router-nv
  table and `drift` checks a hand-written document against one.
- `oapivalid` — compile once, validate many, and never fail.
- `oapidiff` — what changed, and `is_breaking` per audience.

### Known

- **`validate_request` cannot fail.** A request that does not match is
  the answer, not an error; everything genuinely wrong is wrong in the
  document and `compile` says so.
- **Every failure names the request location, the document pointer and
  what was expected** — all of them rather than the first.
- **A catch-all route has no OpenAPI spelling** and `path_template`
  refuses rather than emitting an approximation tools reject.
- **Breaking is asymmetric.** The same edit breaks clients or servers
  depending on which side of the call it is on, so `is_breaking` takes
  an audience.
- **A schema change is reported as breaking**, because deciding whether
  two schemas describe the same values is subtyping and undecidable in
  general.
- **YAML is yaml-nv's job**, in front of `from_json`; 3.0 is refused by
  name rather than partly read.
- **Schemas are written by hand.** utoipa derives them from a
  declaration at compile time; novo-lang's implicit `Serialize` is a
  run-time walk over a value, and the README's table names the four
  things that walk cannot recover.
- **No `@tier(embedded)` claim.** `json.*` is not admitted at that tier.
- **Two dependencies**: schema-nv by range, router-nv by path until it
  is published.
