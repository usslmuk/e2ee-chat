# chat.

End to end encrypted chat. The server stores ciphertext and holds no keys.

Group crypto is MLS (RFC 9420). Next.js

## What it does

- Rooms from a link or a code, joinable either way
- Text and attachments. Filename, type, and size all encrypted
- Emoji reactions per message, grouped and counted
- Delete your own messages. The host can delete the whole chat
- Typing indicators and presence
- Live updates over server sent events, unread counts per room
- No plaintext in the database
- Everything expires

### Cipher

RFC 9420, MLS, and RFC 9180, HPKE. Suite `0x0001`,
`MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519`

Matching RFC 9420:

| id | suite | AEAD | KEM | hash | signature |
|---|---|---|---|---|---|
| 1 | MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519 | AES-128-GCM | X25519 | SHA-256 | Ed25519 |
| 2 | MLS_256_DHKEMX448_AES256GCM_SHA512_Ed448 | AES-256-GCM | X448 | SHA-512 | Ed448 |


```
treekem                      997 pass    62 update paths: node keys, path secret chain, commit secret, tree hash after
tree-math                   6098 pass    tree math at ten tree sizes up to 512 leaves
tree-validation              468 pass    tree hashes and resolutions across fourteen trees
key-schedule                  50 pass    five chained epochs, every derived secret
deserialization               28 pass    variable length vector headers, encode and decode
passive-client-handling-commit 277 pass  26 commits, 20 update paths, commit secrets and epoch authenticators
passive-client-welcome        16 pass    passive join, epoch authenticator, tree hash
secret-tree                   10 pass    handshake and application ratchets, sender data keys
psk_secret                    10 pass    PSK secret chain, one through ten PSKs
tree-operations               10 pass    tree hashes before and after a proposal
crypto-basics                  7 pass    HPKE, HKDF, signing, labelled expansion
welcome                        6 pass    HPKE open, group secrets, GroupInfo, signature
transcript-hashes              2 pass    confirmed and interim transcript chaining, confirmation tag
```

### Elsewhere

- RFC 9180, HPKE. DHKEM(X25519) base mode, verified through the
  `crypto-basics` family.
- RFC 5869, HKDF-SHA256.
- RFC 7748, X25519. RFC 8032, Ed25519.

```
open at message   1: 0.055 ms
open at message  10: 0.056 ms
open at message 100: 0.056 ms
open at message 300: 0.040 ms
```

<details>
<summary>Changelogs</summary>

## 0.1.2

Caching on the paths that run per message, and honest benchmarks.

**Fixed**

- Opening a message re-walked the secret tree on every single message to
  re-derive the sender's leaf secret, and base64-encoded the group id to build a
  cache key it then threw away. Both are stable for the epoch. The sender seed is
  now cached per group per leaf, so open is flat instead of climbing.
- A key package was re-parsed, with two Ed25519 verifications, every time it was
  referenced. A commit parses the same package two or three times, once in
  prepare and again building each Welcome. Parsed packages are now memoised on
  their ref hash, which drops a commit that adds a member from 9.1 ms to 4.8 ms
  and a parse from 2.0 ms to 0.003 ms.
- The update path encrypted the path secret to the committer's own nodes as well
  as everyone else's. The committer already holds that secret. Skipped, per the
  latitude RFC 9420 section 7.6 allows.

**Corrected**

The benchmark table in this readme was wrong. It claimed opening a message grew
from 0.11 ms to 1.38 ms over 300 messages and that a commit cost 14.9 ms.
Measured, opening is flat at 0.055 ms and a commit costs 4.75 ms. The growth
described was real in an older revision but is not now, and nobody re-measured
after the conformance fixes landed. The numbers above are measured, and the
curve primitives are annotated with a native comparison because this machine's
bigint is pathologically slow and the figures are not representative of the
hardware most people will run.

## 0.1.1

The whole MLS layer checked against the working group's published test vectors by a
runner that lives in the repository, and every defect that found corrected.

**Added**

- `pnpm vectors`. Downloads the published vectors, runs them against this code,
  prints a pass count per family, and exits non-zero on any failure. Source in
  `chat/vectors`, runner in `chat/scripts/vectors.mjs`, `esbuild` as the bundler.
  Thirteen of the sixteen published families now run.
- `treekem` covers node keys, the path secret chain, the commit secret and the
  tree hash after an update path, across 62 update paths in eleven trees.
- `passive-client-handling-commit` drives a passive client through a received
  commit end to end, and `transcript-hashes` confirms the confirmed and interim
  transcript chaining.
- Public messages, per RFC 9420 section 6.2, and a Welcome and GroupInfo codec in
  `mls/rfc.ts` that reads and writes the RFC structures.
- TURN support with coturn shared secret credentials, issued per request and
  expiring in an hour. Optional, and off unless both variables are set.
- Content Security Policy carries a per response nonce in production and drops
  `unsafe-inline` from `script-src`.
- Forward secrecy across membership changes. A rolling window of the last 64
  generations per sender is retained and the ratchet root is erased once it fills.
- Invite codes raised to 96 bits. Cursor pagination over message history, ordered
  by document id. Per room blob quota of 64 MB. Unique partial index on room and
  message id, making message send idempotent.

**Fixed, MLS**

Every item here is listed in full under Cryptography above, with the RFC section
it was measured against. In summary: the update path and commit body ordering, the
group secrets PSK list, the zero PSK substitution, the group context extensions
vector, the missing `FramedContentTBS` label, the proposal list packing and
`proposal_type` width, the update proposal payload, dropped update proposals,
`MakeKeyPackageRef`, unmerged leaf handling in resolutions and filtered direct
paths, unmerged leaf persistence, inline `Add` and `PreSharedKey` proposal framing,
the key package encoding, `HPKECiphertext` iteration, `GroupContextExtensions`
proposals, `PSKType` width, `PreSharedKeyID` splitting, the update path resolution
exclusion, the commit signature input range, and the derived scalar clamp in
`derivePair`.

**Fixed, relay**

- Message history dropped messages. The endpoint fetched 300 rows across every
  requested room, kept at most 60 per room, and advanced the cursor past all 300.
  Every discarded row was skipped forever with no error anywhere. The cursor now
  advances past the last row actually returned, so a busy room hands back its
  backlog over successive pages.
- The room quota for blobs only ever went up. It was incremented before the
  insert, never decremented when the insert failed, and never decremented when the
  TTL index deleted the file. A room that used attachments steadily filled with
  phantom quota and eventually refused every upload forever. Usage is now summed
  from the blobs collection on each upload, so expiry reclaims it by construction.
- The two person cap was checked and then the member was pushed, so two
  simultaneous joins both passed. The size check is now part of the atomic update
  filter and the cap is enforced by the database.
- The rate limiter wiped everyone's counters once it held 20000 keys, and the key
  was the leftmost `X-Forwarded-For` value, which the client chooses. Overflowing
  now rejects new keys instead of clearing the table, and the address comes from
  `X-Real-IP` or the rightmost forwarded entry.
- Message bodies accepted 1.4 million characters. The cap is now 200000. The blob
  cap stays at 1.5 MB.
- Every message stored sequence `0`, so the history sort ran on a content hash and
  the 300 message limit applied across all rooms at once.
- Blob uploads accepted 14 MB with no per room quota.
- Commit, welcome and key publication required the room founder, so closing that
  browser mid handshake froze the group permanently.
- An earlier non partial unique index could not be built against existing
  documents, which took the whole app down with `database unreachable`. Connection
  failures were also reduced to that same bare string by an empty catch block.
- `slot` in the peer mesh returned the first peer in the map rather than the
  requested one once at capacity.
- Rate limiting used a fixed window, which let a client double its rate across a
  window boundary.
- `CODE_PEPPER` was loaded and length checked but never used. Invite code hashes
  now use it.

**Fixed, client**

- Private keys were kept in `localStorage` as base64. The identity key, the
  pending offer key, and every group's seed, epoch secrets, held path secrets and
  ratchet secrets were readable by anything with filesystem or devtools access to
  the profile. All of it moved to IndexedDB, read into memory once at startup.
- The hand maintained `BUILD` constant is gone. The storage stamp is the version
  from `package.json`, injected by `next.config.js`, so bumping the version is the
  only thing needed to invalidate old vaults.
- `pack`, `unpack`, `keyedSeal`, `keyedOpen` and `sizeOf` in `lib/blob.ts` were
  never called by anything. The module is gone.
- `chat..png` was 1.1 MB and referenced by nothing. Removed.
- The layout referenced an `apple-icon.png` that was never in `public`, so every
  page load asked for a file that could only 404. The icon is generated from
  `icon.png` now.
- The quick reaction heart was a double encoded UTF-8 sequence and rendered as
  mojibake.

**Changed**

- Rooms are capped at two participants, enforced server side.
- Deleting a message removed it only on the deleting client. The endpoint now
  notifies the room and the merge removes it on both sides.
- `GET /api/v1/msgs` reports whether the per room cap truncated the response, so
  the client only treats an absent id as deleted when the response was complete.
- Ratchet trees are serialized as the RFC's `optional<Node>` vector and round trip
  byte for byte.
- `GroupInfo` uses the RFC wire layout rather than a private node layout.

**Corrected in the documentation**

- Ed25519 is RFC 8032. X25519 is RFC 7748. An earlier revision credited both to
  RFC 7748 and to NIST.
- `CODE_PEPPER` does not sign invite codes. A keyed hash is not a signature.
- Random database field names are obfuscation, not protection.

</details>
