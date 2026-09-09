# jwt-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

JSON Web Tokens — encode and decode, HS256, RS256 and ES256 — with the
unverified state made unusable by construction. Decoding an untrusted
token hands back something whose claims cannot be read; only a
signature check produces a value they can be read from. The `none`
algorithm is not a value this package has. Validation is a policy value
a caller builds and names, rather than nine optional arguments with
security decisions in their defaults.

It is for the service on the receiving end of somebody else's token: an
API gateway, a request handler behind one, a worker validating a job's
credentials, a CLI holding a session.

```
novo pkg add jwt-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use jwttoken
use jwtalg
use jwtclaims
use jwtpolicy

fn check(token: Str, secret: Bytes, now_seconds: Int) -> Result<JwtClaims, JwtError>
    let t   = jwttoken.decode(token)!
    let key = jwtalg.hmac_verifying_key(secret)
    let pol = jwtpolicy.requiring(
                  jwtpolicy.allowing(jwtpolicy.strict(), [Hs256]),
                  "https://auth.example", "api")
    Ok(jwttoken.claims_of(jwttoken.verify(t, key, pol, jwtclaims.instant(now_seconds))!))
```

There is no shorter version of this that skips the check, because
`claims_of` takes a `VerifiedJwt` and nothing but `verify` produces one.

## The load-bearing interface: the unverified state is unusable

Every JWT library offers `decode(token)` and it returns the claims. The
signature check is a separate call, or a flag, or a second argument the
caller may omit — so the shortest path from a string to a `sub` is the
one that checks nothing. That is `jwt.decode(token, verify=False)` in
PyJWT, `jwt.decode` without `WithValidMethods` in Go, and a long tail of
CVEs where an application read a claim out of a token it had not
verified and trusted what it found.

Here there are two token types and only one can be asked for claims:

```novo
pub fn decode(token: Str) -> Result<UnverifiedJwt, JwtError>
pub fn verify(t: UnverifiedJwt, key: JwtVerifyingKey,
              policy: JwtPolicy, now: JwtInstant) -> Result<VerifiedJwt, JwtError>
pub fn claims_of(t: VerifiedJwt) -> JwtClaims
```

`claims_of` takes a `VerifiedJwt`. There is no other way to obtain a
`JwtClaims` from a token, and no way to obtain a `VerifiedJwt` except
from `verify`. So a caller holding claims has been through a signature
check with a key it chose and a policy it wrote — not because it
remembered to, but because it could not write the other program. A
reviewer never has to ask whether something was verified; the type says
so.

**What an unverified token will tell you**, and why that is not a hole:
`header_of`, `alg_of` and `kid_of` work before verification, because
choosing *which key* to verify with requires reading the `kid` — a
token that showed nothing could never be verified at all. What they
return are facts about the token's envelope, and nothing here lets a
caller act on them as claims about the world. In particular the
header's `alg` is never used to select an algorithm.

**The one escape hatch**, named so it cannot be mistaken:
`dangerous_unverified_payload_json` returns the payload as JSON *text*,
so it cannot be passed to anything expecting verified claims. It exists
because introspection endpoints, migrations reading tokens signed by
destroyed keys, and test fixtures genuinely need it, and refusing
outright would send those callers to a hand-rolled base64 split with no
`crit` check and no name a reviewer would notice.

### `none` is unrepresentable

`JwtAlg` has three arms — `Hs256`, `Rs256`, `Es256` — and none of them
is `none`. There is no value a caller or a decoder could construct that
means "unsigned". A token whose header says `alg: none` fails at
`decode` with `UnknownAlgorithm("none")`, before anything else happens.

That is not a check somewhere that a refactor could delete. It is the
absence of an arm.

### The key decides the algorithm, never the token

The other classic attack is algorithm confusion: a token signed HS256
using the RSA *public* key as the HMAC secret, accepted by a verifier
that read the algorithm out of the token's own header. So no function
here takes an algorithm from a header. A `JwtVerifyingKey` carries the
one algorithm it is for, `JwtPolicy` carries the list a caller accepts,
and a token whose header disagrees with the key is
`AlgorithmMismatch`. `JwtSigningKey` and `JwtVerifyingKey` are separate
types, so a public key cannot sign.

### Validation is a value

```novo
pub fn strict() -> JwtPolicy       // accepts nothing until narrowed
pub fn allowing(p: JwtPolicy, algs: [JwtAlg]) -> JwtPolicy
pub fn requiring(p: JwtPolicy, issuer: Str, audience: Str) -> JwtPolicy
```

`strict()` requires an expiry, requires an issuer match, requires an
audience match, and allows no algorithms at all — so a caller must fill
those in before it can verify anything. The awkwardness is the point:
it is the one moment at which somebody looks at what their service
accepts, instead of inheriting `verify_aud=False` from a library
author. `jwtpolicy.check` catches an unusable policy at start-up, where
the message can name the field, rather than per request as a stream of
rejections that look like a client's fault.

## The clock is a parameter

Verifying needs the current time, and this is a `core` package, so it
may not read one. `JwtInstant` arrives as an argument — the shape
`sql-engine-nv` takes with `NowSnapshot`:

```novo
let now = jwtclaims.instant(time.unix_seconds())   // [time], in the host
jwttoken.verify(t, key, policy, now)               // [], here
```

The `[time]` is spent once, in the caller, where it is visible. And it
buys more than the layer rule: a test verifies at any instant without a
fake clock, an audit tool can ask *"was this token valid when it was
presented"* rather than only *"is it valid now"*, and a replay of a
request log gets the same answers it got the first time. A library that
read the clock itself could do none of those.

## The layer, and why

`core` — no effects. Nothing here opens a file, resolves a name, or
fetches a key. A JWKS fetch is `[net]` work and belongs in a `host`
package on top of this one; key *selection* is here, as
`jwtalg.key_for_kid`, because choosing between keys you already hold is
arithmetic.

## Status: two algorithms have no primitives to be implemented with

The interface publishes HS256, RS256 and ES256, and only one of them
can be implemented against the grid as it stands today. This is stated
here rather than discovered at the first body:

- **HS256 is ready.** `crypto-nv` supplies `hmac_sha256`.
- **ES256 needs ECDSA.** `p256-nv` 0.0.1 has key construction,
  validation and ECDH — no `ecdsa_sign`, no `ecdsa_verify`, and nothing
  else on the grid has them either. The keys are p256-nv's types
  already, so the gap is two functions in that package.
- **RS256 needs RSA, and there is no RSA anywhere.** No Orbit package
  provides it, and none is planned yet. The signing and verifying keys
  hold PKCS#1 DER as bytes rather than a structured type for exactly
  that reason: there is no package to hold the structure.

**And a toolchain defect defeats the load-bearing design today.** A
qualified call `m.f(x)` does not check its argument types, so
`jwttoken.claims_of(unverified)` — and `jwttoken.claims_of(42)` —
compile, and fail at run time. The two-token design is correct and the
compiler will enforce it once that is fixed; until then it is enforced
by review, and this is where that is written down rather than assumed.
The defect is filed against the toolchain as
`qualified-cross-module-calls-do-not-check-argument-types`.

## The reference implementation

Rust's `jsonwebtoken` for the shape of a validation value rather than a
pile of arguments, and Go's `golang-jwt` for the key-function pattern
that `verify_with_keys` replaces with a plain list scan. PyJWT is the
reference for what *not* to do, in a specific and useful way: its
`verify=False` and its algorithm-from-the-header defaults are the two
mistakes this package's types exist to make unwritable. RFC 7519, RFC
7515 and RFC 7518 are the oracle, and the `jwt.io` and Auth0 conformance
vectors are the test set.

Deliberately not ported: JWE, which is a different specification and a
different package; JWKS fetching and caching, which is `[net]` and
belongs in a `host` package; a JSON value type of this package's own —
`payload_json` is text, so the caller uses the decoder it already has
and this package's type names cannot collide with `serde-nv`'s.

## Status

Every function is `todo()`. `novo test --isolate` runs the API suite
and every assertion reaches `not implemented: jwt-nv.<module>.<fn>`.

| module | public types | functions | implemented |
| --- | --- | --- | --- |
| `jwtalg` | `JwtAlg`, `JwtSigningKey`, `JwtVerifyingKey` | 14 | no |
| `jwtclaims` | `JwtInstant`, `JwtHeader`, `JwtClaims` | 17 | no |
| `jwttoken` | `UnverifiedJwt`, `VerifiedJwt` | 13 | no |
| `jwtpolicy` | `JwtPolicy` | 10 | no |
| `jwterror` | `JwtError` (+ `impl Error`) | 3 | no |

Ten public types, 57 public functions and one trait impl.

## Licence

Apache-2.0.
