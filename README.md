# http-signatures

HTTP Signatures for [Quarz](https://quarz.dev), written in Quarz over its
standard library: signing an HTTP request with the `Signature` header of
[draft-cavage-http-signatures-12](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures-12),
verifying a request signed so, and the [Digest] header (RFC 3230) of the
body it covers.

```toml
# Cog.toml of an application
[dependencies]
HTTPSignatures = { git = "https://github.com/quarz-lang/http-signatures", tag = "v0.1.0" }
```

The API reference of the main branch is at <https://http-signatures.api.quarz.dev>.
The design and its decisions are in issue #1.

## Signing a request

```crystal
key = Crypto::RSA::PrivateKey.from_pem(File.read("private.pem"))
req = HTTP::Client::Request.post("https://example.com/orders", body: json, content_type: "application/json")
signed = HTTPSignatures.sign(req, key_id: "client-1", key: key)
HTTP::Client.default.request(signed)
```

The key is held natively by quarz: parsed once, never a `Bytes` in the
value, cleared when the last reference goes. `key.wipe` ends it for
every holder at once (a `sign` with it then raises
`Bytes::Secret::WipedError`). `from_pem` also takes a `Bytes::Secret`,
for a PEM that should not stay in memory as a `String`:

```crystal
key = Crypto::RSA::PrivateKey.from_pem(Bytes::Secret.of(File.read_bytes("private.pem")))
# ... sign with it ...
key.wipe
```

`sign` gives a copy of the request with a `Signature` header. By default
it covers `(request-target) host date digest`, adding [Date] and
[Digest] when the request has none; `(request-target)` and `host` are
what the client sends ([HTTP::Client::Request#target], `#host`).
`algorithm:` is `"rsa-sha256"` (the default) or `"hs2019"`.
`created:`/`expires:` write those parameters whenever they are given;
covering `(created)` or `(expires)` needs hs2019.

## Verifying a request

```crystal
handler = HTTP.handler do |req|
  begin
    v = HTTPSignatures.verify(req) { |key_id| key_for(key_id) }   # a Crypto::RSA::PublicKey or nil
    HTTP::Server::Response.text("hello #{v.key_id}")              # v.body: the body, read and checked
  rescue HTTPSignatures::Error => e
    HTTP::Server::Response.text(e.message, 401)
  end
end
```

`verify` checks, in order: the parameters (from `Signature`, or
`Authorization: Signature`), the algorithm, the covered names, that
`required:` names (by default `(request-target) host date`) are covered,
the time ([Date], `created`, `expires` within `max_skew:`, five minutes),
the signature with the key the block gives, and then the body against
the covered [Digest] (a body no [Digest] covers is refused unless
`require_digest: false`). Each failure raises a subclass of
[HTTPSignatures::Error] whose message says why and quotes nothing the
peer sent.

`lenient: true` (off by default) also takes forms the draft does not
allow and signers in use send: whitespace or the URL-safe alphabet in
the signature's Base64, a `created` with a fraction (dropped), a bare
value running to the next comma (trimmed), empty entries after a comma,
and a parameter given twice, of which the **first** is used. Careful
with the last one: anything else reading the same header (a proxy, a
log, another verifier) must take the same one, or it sees another
`keyId` or `headers` than the one verified; the default refuses a
parameter given twice. The signing string is built from the parameters
as used (the first of a repeated one, a bare value trimmed, `created`
as written).

The block is given the peer's `keyId` before anything is verified: a
block that fetches a key by it must check it first (a URL to fetch is a
request the peer chose).

An application that reads the body itself first (its own limit, a key
found from the body, the body of a request that fails kept) gives it:
`verify(req, body: bytes) { ... }` reads nothing and checks the [Digest]
against `bytes`. `verify_signature(req.method, req.target, req.headers)
{ ... }` verifies the signature alone and gives a `Signed` (`key_id`,
`algorithm`, `headers`), for the body checked apart
(`HTTPSignatures::Digest.matches?`).

The body is read only once the signature has verified: after
`BadSignature`, a second `verify` of the same request with a refreshed
key (a peer that rotated its key) works.

## What is supported

- Keys: RSA ([Crypto::RSA]). `rsa-sha256` is RSASSA-PKCS1-v1_5 with
  SHA-256; `hs2019` is that, or RSASSA-PSS with SHA-512 (verifying
  accepts either; signing writes the first unless `hs2019_pss: true`).
  A signature naming no algorithm is verified as `hs2019`.
- `(request-target)`, `(created)`, `(expires)` and any header; repeated
  headers joined by `, `; at most 32 covered names. Without a `headers`
  parameter, `date` is covered for `rsa-sha256` and `(created)` for
  `hs2019`.
- [Digest] with `SHA-256` (the name in any case, among other entries).
- The draft's Appendix C test vectors, checked byte for byte.

Not supported: HMAC, ECDSA and Ed25519 keys; `rsa-sha1`; `(created)` or
`(expires)` with `rsa-sha256` (the draft forbids them there); other
[Digest] algorithms and RFC 9530's `Content-Digest`; RFC 9421 (HTTP
Message Signatures), the draft's standard successor; signing a streamed
body's [Digest]; signing responses.

## License

MIT No Attribution: see [LICENSE](LICENSE).
