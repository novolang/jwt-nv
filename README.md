# jwt-nv

A JSON Web Token (JWT) is a compact, URL-safe way of representing
claims to be transferred between two parties, specified in
[RFC 7519](https://www.rfc-editor.org/rfc/rfc7519). A signed one is a
JSON Web Signature (JWS), specified in
[RFC 7515](https://www.rfc-editor.org/rfc/rfc7515), whose algorithms
are in [RFC 7518](https://www.rfc-editor.org/rfc/rfc7518). This package
reads and writes signed JWTs in novo-lang, and gives the unverified
form a type whose claims cannot be read.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a JWT is

A signed JWT is three parts separated by dots. Each part is base64url
with no padding. The first is the **header**, a JSON object naming the
signature algorithm in its `alg` member and, usually, a **key
identifier** in `kid`. The second is the **payload**, a JSON object of
**claims**. The third is the **signature** over the first two joined by
a dot, which RFC 7515 section 5.1 calls the **signing input**.

A claim is one member of the payload. RFC 7519 section 4.1 registers
seven, and each is optional.

| Claim | Meaning |
| --- | --- |
| `iss` | Who issued the token |
| `sub` | Who or what the token is about |
| `aud` | Which service the token is for |
| `exp` | The instant after which it must not be accepted |
| `nbf` | The instant before which it must not be accepted |
| `iat` | When it was issued |
| `jti` | A unique identifier for this token |

`exp`, `nbf` and `iat` are numbers of seconds since the Unix epoch
(section 2's *NumericDate*).

The algorithms here are the three RFC 7518 section 3.1 names most
deployments use.

| Algorithm | What signs | Key |
| --- | --- | --- |
| HS256 | HMAC with SHA-256 | one shared secret, at least 32 bytes |
| RS256 | RSASSA-PKCS1-v1_5 with SHA-256 | an RSA key pair |
| ES256 | ECDSA on P-256 with SHA-256 | a P-256 key pair |

The same section registers `none`, meaning an unsigned token. This
package has no value for it, so a token whose header says `alg: none`
is refused when it is decoded.

Verifying a token needs the current time, and this package declares no
effects, so the time arrives as a `JwtInstant` argument.

## Install

```
novo pkg add jwt-nv
```

## Example

```novo
use std.bytes
use jwtalg
use jwtclaims
use jwtpolicy
use jwttoken

fn main() [io]
    // The shared secret this service verifies HS256 tokens with.
    let key = jwtalg.hmac_verifying_key(bytes.from_str("a 32-byte secret goes here......"))

    // What this service accepts: HS256 only, from one issuer, for one
    // audience. `strict()` accepts nothing until it is narrowed.
    let policy = jwtpolicy.requiring(
                     jwtpolicy.allowing(jwtpolicy.strict(), [Hs256]),
                     "https://auth.example", "api")

    // The current time, read by the caller and passed in.
    let now = jwtclaims.instant(1789000000)

    // Split the token. Its claims cannot be read yet.
    match jwttoken.decode("eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZGEifQ.c2ln")
        Err(e) => println("not a token: ${e.message()}")
        Ok(t)  =>
            // Check the signature and the policy. Only this call can
            // produce the value that `claims_of` accepts.
            match jwttoken.verify(t, key, policy, now)
                Err(e) => println("refused: ${e.message()}")
                Ok(v)  => println("the subject is ${jwttoken.claims_of(v).sub}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: jwt-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `jwterror` | Every refusal, and whether each is safe to report to the presenter of the token. |
| `jwtclaims` | An instant in Unix seconds, the header, the claims, and the builders and predicates over them. |
| `jwtalg` | The three algorithms, the signing and verifying key types, and the choice of key by identifier. |
| `jwtpolicy` | What a service accepts, as a value: the algorithms, the issuer, the audience, the leeway and the lifetime cap. |
| `jwttoken` | Decoding, verifying, reading a verified token's claims, and encoding. |

## How to choose an entry point

**`jwttoken.decode` splits a token and reads its envelope.** It answers
an `UnverifiedJwt`, whose header, algorithm and key identifier can be
read and whose claims cannot.

**`jwttoken.verify` takes one key.** `verify_with_keys` takes a list
and chooses by the token's `kid`. Use the second when a service holds
several keys, such as across a key rotation.

**`jwtpolicy.strict()` is the policy to start from.** It accepts no
algorithm at all, so it must be narrowed with `allowing` and
`requiring` before it can verify anything. `permissive()` exists for a
test and says so in its name.

**`jwttoken.encode` produces a token.** `signing_input` and
`attach_signature` are the two halves for a caller signing with a key
this package cannot hold, such as one in a hardware security module.

## The rules a user needs

1. **Claims can only be read from a verified token.**
   `jwttoken.claims_of` takes a `VerifiedJwt`, and `jwttoken.verify` is
   the only function that produces one. A program that reads a `sub` has
   been through a signature check against a key it chose and a policy it
   wrote. The compiler refuses the other program.
2. **The key decides the algorithm, never the token.** A
   `JwtVerifyingKey` carries the one algorithm it is for, and a token
   whose header disagrees is `AlgorithmMismatch`. This is the refusal
   that stops a token signed HS256 with an RSA public key as the secret
   from being accepted.
3. **`alg: none` is refused at `decode`**, as
   `UnknownAlgorithm("none")`. `JwtAlg` has three arms and none of them
   means unsigned.
4. **`JwtSigningKey` and `JwtVerifyingKey` are separate types**, so a
   public key cannot sign.
5. **An unverified token will tell you its header, its algorithm and
   its `kid`.** Reading the `kid` is how a caller chooses which key to
   verify with, so a token that showed nothing could never be verified.
   None of those is a claim about the world, and the header's `alg` is
   never used to select an algorithm.
6. **`dangerous_unverified_payload_json` answers text, not claims.**
   An introspection endpoint, a migration reading tokens signed by a
   destroyed key and a test fixture each need the payload of a token
   they cannot verify. The result is JSON text, so it cannot be passed
   anywhere that expects verified claims, and the name is what a
   reviewer sees.
7. **`strict()` accepts nothing until it is narrowed.** It requires an
   expiry, requires an issuer match, requires an audience match, and
   allows no algorithm. Narrowing it is the one moment at which
   somebody states what the service accepts.
8. **Run `jwtpolicy.check` at start-up.** It answers the fault in an
   unusable policy, naming the field. Left to run time, the same faults
   arrive as a stream of rejections that look like a client's mistake.
9. **The current time is an argument.** `jwtclaims.instant` takes Unix
   seconds the caller read. A test verifies at any instant with no fake
   clock, an audit tool can ask whether a token was valid when it was
   presented, and a replay of a request log gives the same answers it
   gave the first time.
10. **`crit` is refused, not ignored.** A header naming critical
    extensions this package does not understand is
    `UnsupportedCritical`. RFC 7515 section 4.1.11 requires that,
    because ignoring the member is how a security-relevant extension
    gets skipped.
11. **An HS256 key shorter than 32 bytes is refused**, at signing as
    well as at verifying, so the tokens never exist. RFC 7518 section
    3.2 sets that length.
12. **Some refusals are safe to report to the presenter and some are
    not.** `jwterror.is_client_safe` answers which. Telling a client
    "bad signature" rather than "invalid token" builds a forgery
    oracle. Telling it that its token expired is what triggers a
    refresh instead of a retry loop.
13. **`jwterror.is_configuration_fault` separates a deployment's own
    mistakes** from anything a client did. `PolicyAcceptsNothing`,
    `PolicyIncomplete` and `WeakKey` are that set.
14. **An `nbf` that is present is always honoured**, whether or not the
    policy requires one.
15. **`max_lifetime_seconds` caps `exp - iat`.** A token with a
    ten-year lifetime passes every other check and is still a
    credential nobody can revoke.

## What `strict()` sets

| `JwtPolicy` field | `strict()` |
| --- | --- |
| `allowed_algs` | empty, so nothing is accepted |
| `require_iss`, `require_aud`, `require_exp` | true |
| `issuer`, `audience` | empty, so both must be supplied |
| `require_nbf`, `require_sub` | false |
| `leeway_seconds` | 0 |
| `max_lifetime_seconds` | 0, meaning no cap |

`require_nbf` is false because most issuers set no `nbf`.
`require_sub` is false because a token about a service rather than a
person has none.

## Which algorithms can be implemented today

The interface publishes three algorithms. Only one of them has a
primitive on the registry to be implemented with, and that is stated
here rather than discovered at the first body.

| Algorithm | What it needs | Available |
| --- | --- | --- |
| HS256 | `hashing.hmac_sha256` from crypto-nv | yes |
| ES256 | `ecdsa_sign` and `ecdsa_verify` over P-256 | no. p256-nv 0.0.2 has key construction, validation and ECDH, and neither signing function |
| RS256 | RSASSA-PKCS1-v1_5 | no. There is no RSA package on the registry |

The RSA signing and verifying keys hold PKCS#1 DER as bytes rather than
a structured type, because there is no package to hold the structure.
The ECDSA keys are already p256-nv's own types.

## Timing behaviour

No claim is made here beyond what the primitives underneath provide.
HS256 verification is an HMAC comparison, and the comparison that
matters is crypto-nv's `digest.ct_eq`, which is constant-time. The
other two algorithms have no implementation yet, so there is nothing to
claim about them.

## What is not included

- **JWE, the encrypted form of a JWT.** It is a different
  specification, RFC 7516, and a different package. Everything here is
  a signed token, whose payload anybody holding it can read.
- **Fetching a JWKS.** Retrieving a key set over the network costs
  `[net]`, and this package declares no effects. Choosing between keys
  a caller already holds is here, as `jwtalg.key_for_kid`.
- **A JSON value type of this package's own.** `payload_json` and
  `JwtClaims.payload_json` are text, so a caller uses the JSON decoder
  it already has.
- **A clock.** See rule 9.
- **RSA and ECDSA implementations.** See the table above.

## Related packages

- [paseto-nv](https://novo-lang.org/packages/paseto-nv) is PASETO, a
  token format with no algorithm field at all: the version and the
  purpose are a fixed prefix and the key carries its own purpose. Take
  it for a new system. Take this package for tokens somebody else
  already issues.
- [oauth2-nv](https://novo-lang.org/packages/oauth2-nv) is the client
  side of OAuth 2 and OpenID Connect, which is where most of these
  tokens come from. An OpenID Connect identity token is a JWT.
- [cookie-nv](https://novo-lang.org/packages/cookie-nv) signs and
  encrypts cookie values. Take it when the value never leaves this
  service. Take this package when another party must verify the token.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the
  HMAC under HS256. This package depends on it.
- [p256-nv](https://novo-lang.org/packages/p256-nv) supplies the P-256
  key types used by ES256. This package depends on it.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is the
  base64url encoding the three parts use.

## Tests

```bash
novo test tests/jwttoken_tests.nv    # decoding, verifying, and encoding
novo test tests/jwtpolicy_tests.nv   # what a policy accepts and refuses
novo test tests/jwtclaims_tests.nv   # the registered claims and the instants
novo test tests/jwtcover_tests.nv    # every public function is reached
```

The normative sources are RFC 7519 for the claims, RFC 7515 for the
signature and the `crit` rule, and RFC 7518 for the algorithms and the
minimum HMAC key length. The vectors are the conformance tokens
published by jwt.io and by Auth0. The reference implementations are the
Rust crate `jsonwebtoken`, for a validation value rather than a pile of
arguments, and Go's `golang-jwt`, whose key function this package
replaces with a plain list scan.

The suite asserts that `alg: none` is refused at decode, that a token
signed with one algorithm and verified with a key for another is
`AlgorithmMismatch`, that `strict()` accepts nothing until it is
narrowed, that an expired token reports both instants, that a `crit`
this package does not understand is refused, and that an HS256 key
under 32 bytes is refused at signing.

The tests compile today and fail at run, each on the
`not implemented: jwt-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `jwtalg.JwtAlg`, `.JwtSigningKey`, `.JwtVerifyingKey` and the other types | the types are declared |
| `jwterror.is_client_safe`, `.is_configuration_fault`, `.code`, `JwtError.message` | no |
| `jwtclaims.instant`, `.unix_seconds`, `.plus` | no |
| `jwtclaims.header`, `.header_with_kid`, `.claims` | no |
| `jwtclaims.with_iss`, `.with_sub`, `.with_aud`, `.with_exp`, `.with_nbf`, `.with_iat`, `.with_jti`, `.with_payload` | no |
| `jwtclaims.is_expired`, `.is_not_yet_valid`, `.has_audience` | no |
| `jwtalg.alg_name`, `.alg_named`, `.is_symmetric` | no |
| `jwtalg.hmac_key`, `.hmac_verifying_key`, `.rsa_signing_key`, `.rsa_verifying_key`, `.ec_signing_key`, `.ec_verifying_key` | no |
| `jwtalg.signing_alg`, `.verifying_alg`, `.with_kid`, `.kid_of`, `.key_for_kid` | no |
| `jwtpolicy.strict`, `.permissive`, `.allowing`, `.requiring` | no |
| `jwtpolicy.with_leeway`, `.with_max_lifetime`, `.requiring_subject`, `.requiring_nbf` | no |
| `jwtpolicy.accepts_alg`, `.check` | no |
| `jwttoken.decode`, `.verify`, `.verify_with_keys`, `.claims_of` | no |
| `jwttoken.verified_header_of`, `.verified_alg_of`, `.header_of`, `.alg_of`, `.kid_of` | no |
| `jwttoken.dangerous_unverified_payload_json` | no |
| `jwttoken.encode`, `.signing_input`, `.attach_signature` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
