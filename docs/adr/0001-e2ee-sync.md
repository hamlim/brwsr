# ADR 0001: End-to-end encrypted sync

Status: proposed. Owner: Matt Hamlin. Date: 2026-10-27.

## Decision

All synced user data is encrypted on device with keys the service never holds. Sync uses
whole-item writes with optimistic version checks. The service orders and stores; it never
merges, reads, or dedupes by content.

## Keys

| Key | Made where | Stored where | Use |
|---|---|---|---|
| Device keypair (X25519 + Ed25519) | on each device | Secure Enclave where available, else Keychain, non-exportable | enrollment, wrapping, signing device metadata |
| Account key (256-bit) | first device | Keychain, wrapped by the device key | root of all data keys |
| Data key per kind group | HKDF(account key, kind group) | derived on use | `items`, `cookies`, `credentials` are separate groups |
| Space key (P2) | per Space, HKDF or random | wrapped by account key | sharing a Space without sharing the account |
| Recovery phrase | first device, shown once | user's responsibility | BIP39 24 words, 256 bits, built-in checksum, wraps the account key |

Rotation: a new account key can be made at any time. Items are re-encrypted lazily on write
and in a background pass. Rotation protects future data only. A removed device keeps what it
already had.

## Enrollment

1. New device creates its keypair and shows a 6-digit code plus a QR that carries the code
   and the public key fingerprint.
2. Old device scans the QR or the user types the code. The two devices run a PAKE (candidate:
   CPace; alternative: OPAQUE) over a relay channel through the Worker. The relay carries
   opaque bytes and cannot learn the code or the session key.
3. Old device verifies the new device's public key fingerprint inside the PAKE-protected
   channel, then sends the account key wrapped to that public key.
4. Old device signs a device record `{ deviceId, publicKey, enrolledBy, time }` with its
   Ed25519 key and submits it to the DO. Other devices verify the chain back to the first
   device.
5. Codes expire after 5 minutes. Three wrong attempts cancel the session.

A substituted public key fails step 3 because the fingerprint does not match the one shown on
the new device's screen. The service cannot inject a device because it cannot sign a device
record with any enrolled device's key.

Recovery: the user types the 24 words on a clean device. The device unwraps the account key
from the recovery blob stored in the DO, enrolls itself, and self-signs its device record,
marked `recovered`. Other devices show a warning on next sync.

## Op format

```
{
  opId:        UUIDv7, unique per op, idempotent
  deviceId:    sender
  itemId:      UUIDv7
  kind:        string
  parentId:    UUIDv7 | null
  order:       fractional key string, tie-break by itemId
  baseVersion: integer, 0 for create
  del:         boolean
  keyVersion:  integer, which account key
  schema:      integer, field record schema version
  nonce:       24 bytes
  ciphertext:  AEAD(fields JSON)
}
```

AEAD: XChaCha20-Poly1305 (libsodium on all three platforms), fallback AES-256-GCM via
CryptoKit if libsodium is rejected in the M2 trial. Associated data is the canonical
encoding of every field above except `nonce` and `ciphertext`, plus `accountId`. A deletion
is an op with `del: true` and empty fields, still authenticated. The service rejects nothing by
content, but every client rejects any op whose AEAD tag fails and reports it.

`ciphertext` is the **complete** field record for the item. There are no partial updates.

## Server rule

```
on op:
  if opId seen: return the stored ack
  item = items[itemId]
  current = item ? item.version : 0
  if op.baseVersion != current: reject { stale: true, current: item }
  commit op to oplog with seq = next(), item.version = current + 1
  ack { opId, seq, version }
  fan out to other sockets
```

Commit and ack happen inside one DO transaction, so an ack always means durable.

## Client rule

- Writes: one SQLite transaction updates `items` and inserts into `outbox`. The sender
  retries the outbox in order with backoff. Entries leave the outbox on ack only.
- Stale rejection: client takes the returned current record, re-applies its own field changes
  on top (it knows which fields it changed because it keeps the pre-edit copy in the outbox
  row), and resubmits with the new `baseVersion`. If the item is deleted on the server, the
  client drops the edit and shows an undo toast for 10 seconds.
- Receive: apply any op with `version > local.version`. Never merge. Never apply an older
  version.
- Cursor: client stores `lastSeq`. On connect it sends `lastSeq`. If the server replies
  `expired`, the client takes a snapshot, drops outbox rows for items not in the snapshot,
  rebases the rest, and continues.
- Gaps: a `seq` or per-item `version` gap triggers a snapshot and a warning counter.

## What the service sees

User id, device ids and public keys, signed device records, item ids, kinds, parent ids,
order keys, versions, key versions, schema versions, sizes, timestamps, socket activity.
It does not see titles, URLs, field contents, cookie values, credentials, or Boost code.

## Revocation

Remote sign-out: the DO marks the device revoked, closes its sockets, and rejects its ops.
The Worker deletes its refresh token. Access JWTs live 10 minutes, so the worst-case window
for a non-DO endpoint is 10 minutes; DO endpoints reject within one message. Target: 60 s.

## Limits we accept

- Metadata and timing leak to the service.
- A malicious service can serve one device a consistent but stale view. Clients detect gaps
  but not a frozen view. A signed "latest seq" gossip between devices is a P2 mitigation.
- Losing every device and the recovery phrase loses synced data. The account survives.

## Unknowns and the test that settles each

| Unknown | Test |
|---|---|
| CPace or OPAQUE library availability in Swift and Workers | build the enrollment demo in M2 with both; pick the one that passes on iOS, macOS, and Workers |
| libsodium in a Worker | run the AEAD test vectors in the DO; if blocked, use WebCrypto AES-GCM everywhere |
| Rebase cost on large items | M2 soak with 100k items and 20 devices |
| Secure Enclave for X25519 | macOS and iOS trial; fall back to Keychain non-exportable keys |
