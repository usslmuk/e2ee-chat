<div align="center">
  <h1>chat.</h1>
  <p>End-to-end encrypted chat built with Next.js and MLS (RFC 9420)</p>
</div>

Group encryption uses HPKE, X25519, AES-128-GCM, SHA-256 and Ed25519. Supports messages, attachments, reactions and live updates. No plaintext is stored in the database.

## Tests

| Test | Passed |
|---|---:|
| TreeKEM | 997 |
| Tree math | 6098 |
| Tree validation | 468 |
| Key schedule | 50 |
| Deserialization | 28 |
| Passive client handling commit | 277 |
| Passive client welcome | 16 |
| Secret tree | 10 |
| PSK secret | 10 |
| Tree operations | 10 |
| Crypto basics | 7 |
| Welcome | 6 |
| Transcript hashes | 2 |

Tested against the official MLS RFC 9420 test vectors.

## Benchmarks

| Benchmark | Result |
|---|---:|
| Open message (1) | 0.055 ms |
| Open message (10) | 0.056 ms |
| Open message (100) | 0.056 ms |
| Open message (300) | 0.040 ms |
