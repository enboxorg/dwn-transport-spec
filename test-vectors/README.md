# DWN Transport Protocol Test Vectors

Language-agnostic, deterministic JSON conformance test vectors for the
[DWN Transport Protocol Specification](https://dwn-transport-spec.pages.dev).

## Files

| File | Coverage |
|---|---|
| [`http-framing.json`](./http-framing.json) | Exact hex frames for `application/vnd.enbox.dwn-rpc` HTTP request bodies (`0x00` envelope-only, `0x01` with UTF-8 data, `0x01` complete 0-byte `RecordsWrite`, `0x00` dataless initial write), 8 invalid frames rejected with HTTP `400` / JSON-RPC `-50400`, and `Content-Type` header matching |
| [`proof-of-work.json`](./proof-of-work.json) | Exact Terms of Service SHA-256 hash, UTF-8 preimage concatenation (`challengeNonce \|\| responseNonce \|\| compactRegistrationDataJson`), and passing vs. insufficient SHA-256 solutions (with and without `termsOfServiceHash`) |
