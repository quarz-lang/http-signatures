# http-signatures

HTTP Signatures for [Quarz](https://quarz.dev), written in Quarz over its
standard library: signing an HTTP request with the `Signature` header of
[draft-cavage-http-signatures-12](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures-12),
verifying a request signed so, and the `Digest` header (RFC 3230) of the
body it covers.

```toml
# Cog.toml of an application
[dependencies]
HTTPSignatures = { git = "https://github.com/quarz-lang/http-signatures", tag = "v0.1.0" }
```

The API reference of the main branch is at <https://http-signatures.api.quarz.dev>.

Work in progress: the design is issue #1. Nothing is released yet.

## License

MIT No Attribution: see [LICENSE](LICENSE).
