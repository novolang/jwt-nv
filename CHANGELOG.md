# Changelog

All notable changes to jwt-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.5 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves crypto-nv 0.1.1 to 0.1.6.  crypto-nv 0.1.1 writes
  into lists through names that are not declared `var`, which novo 0.11
  refuses (E2038), so this package did not build with novo 0.11 against
  it.  No requirement in the manifest changed.

## 0.0.4 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.3 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.2 — 2026-09-09

- **Dependencies are registry ranges**, not paths: the interface release 0.0.1 shipped a manifest whose dependencies pointed at sibling directories that exist only in the monorepo, so a consumer resolved the closure and then could not load the dependency.  No signature changed.

## [0.0.1] — 2026-09-09

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `jwttoken` — the load-bearing interface. `UnverifiedJwt` and
  `VerifiedJwt` are two types and only the second can be asked for
  claims: `claims_of` takes a `VerifiedJwt`, nothing but `verify`
  produces one, and there is no other route from a token to a
  `JwtClaims`. So the shortest path from a string to a `sub` goes
  through a signature check, and a reviewer never has to ask whether
  one happened. The header is readable unverified because choosing a
  key requires it; the claims never are.
  `dangerous_unverified_payload_json` is the one escape hatch, named to
  be seen and returning text rather than claims.
- `jwtalg` — `JwtAlg` with three arms and no `none`, so the famous
  forgery is closed by the absence of a value rather than by a check.
  `JwtSigningKey` and `JwtVerifyingKey` are separate types, each
  carrying its own algorithm, so a public key cannot sign and a token
  whose header disagrees with the key is `AlgorithmMismatch` — which is
  what closes algorithm confusion.
- `jwtpolicy` — validation as a value a service builds once and names,
  instead of nine optional arguments with security decisions in their
  defaults. `strict()` accepts nothing until narrowed; `check` catches
  an unusable policy at start-up where the message can name the field.
- `jwtclaims` — the seven registered claims as fields and everything
  else as the payload's raw JSON text, so this package declares no JSON
  value type to collide with anyone's. `JwtInstant` is the clock, as a
  parameter: a `core` package may not read one, and the by-product is
  that a test verifies at any instant and an audit tool can ask what
  was valid when.
- `jwterror` — seventeen reasons, with `is_client_safe` dividing the
  ones a presenter may be told from the ones that would tell an
  attacker its forgery was detected, and `is_configuration_fault`
  separating a 500 from a 401.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  jwt-nv.<module>.<fn>`.
- **ES256 and RS256 have no primitives to be implemented with.**
  `p256-nv` 0.0.1 has no `ecdsa_sign`/`ecdsa_verify`, and there is no
  RSA package on the grid at all. HS256 is ready over `crypto-nv`. The
  README's Status section carries the detail.
- **A compiler defect defeats the two-token design today.**
  `bugs/type-system-meta/qualified-cross-module-calls-do-not-check-argument-types.md`:
  a qualified call does not check its argument types, so
  `jwttoken.claims_of(unverified)` compiles and segfaults. The design
  is correct and the compiler will enforce it once that is fixed.

### Design notes for 0.0.4

The earlier README said that a qualified cross-module call did not
check its argument types, so `jwttoken.claims_of(42)` compiled and
failed at run time, and that the two-token design was therefore
enforced by review rather than by the compiler. That defect is fixed in
the toolchain this release is checked against: the same call is now
refused with `E2001 type error: argument 1 of 'jwttoken.claims_of':
expected VerifiedJwt but got Int`. The README states the guarantee
without the caveat.

PyJWT is the reference for what not to do: its `verify=False` and its
reading of the algorithm out of the token's own header are the two
mistakes this package's types make unwritable.
