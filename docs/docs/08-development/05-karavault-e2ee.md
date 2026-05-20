# Karavault E2EE Design Notes

Status: initial design seed for issues #1 and #2.

Karavault is a fork-only privacy variant of Karakeep. The E2EE design should
assume that the server, database, object storage, queue backend, search index,
server-side backups, logs, and admin tools are not trusted with private
bookmark data.

## References

The primary reference model is Ente:

- Ente architecture: https://ente.com/architecture/
- Ente SRP write-up: https://ente.com/blog/ente-adopts-secure-remote-passwords/

The parts to copy conceptually are:

- client-generated root key material,
- password-derived key encryption key,
- server-stored encrypted key material,
- recovery key,
- hierarchical collection/item data keys,
- public/private key material for future sharing.

Karavault does not need to copy Ente auth in the MVP. NextAuth can remain the
server authentication layer while the vault passphrase remains a separate
client-side encryption secret.

## Threat Model

Vault mode should protect these values from server-side plaintext access:

- bookmark title, note, summary, URL, source URL, link title, description,
  author, publisher, image URL, favicon, HTML/plain text content,
- text bookmark body,
- asset content, extracted text, metadata, original filename, source URL,
- tag names,
- list names, descriptions, and smart-list queries,
- highlight text and notes,
- reading progress anchors when they can reveal text.

The MVP may still expose operational metadata:

- user id, item id, object id,
- created/modified timestamps,
- item type,
- archived and favourited flags,
- approximate object size and storage usage,
- request timing, counts, and feature usage.

Anything outside this allowed metadata list should be treated as private until
there is a deliberate design decision.

## MVP Scope

The MVP is web-first and new-vault-first.

Required:

- create and unlock vault key material in the browser,
- store only encrypted private bookmark/list/tag/highlight/asset payloads,
- render encrypted records after unlock,
- clear decrypted keys and data on lock/logout,
- disable server-side private-content crawler, search, AI, public list, RSS,
  webhook, and admin-debug leakage paths.

Deferred:

- mobile, browser extension, CLI, and MCP vault clients,
- encrypted multi-user sharing,
- complete metadata hiding,
- server-readable full text search,
- server-side AI over private data,
- client-driven migration from existing plaintext records.

## Key Hierarchy

Use versioned, random 256-bit keys.

- `masterKey`: generated on the client during vault setup. Never leaves the
  client unencrypted.
- `keyEncryptionKey`: derived locally from the vault passphrase and stored KDF
  params. It wraps `masterKey`.
- `recoveryKey`: generated locally and shown once for user backup. It can unwrap
  `masterKey` during recovery.
- `vaultKey`: default personal collection key wrapped by `masterKey`.
- `bookmarkKey`: per-bookmark key wrapped by `vaultKey`.
- `assetKey`: per-asset key wrapped by `bookmarkKey` or `vaultKey`.
- `publicKey` and `encryptedPrivateKey`: created during setup so encrypted
  sharing can be added later without redesigning account crypto.

The first implementation should keep passphrase auth and server auth separate.
SRP can be revisited after vault mode works.

## Envelope Format

Encrypted payloads should be self-describing and authenticated with associated
data.

```json
{
  "version": 1,
  "alg": "xchacha20poly1305",
  "keyId": "vault-or-item-key-id",
  "nonce": "base64url",
  "ciphertext": "base64url",
  "aad": {
    "type": "bookmarkPayload",
    "recordId": "bookmark-id",
    "schema": 1
  }
}
```

Associated data should bind ciphertext to record id, payload type, owner id
where useful, and schema version. Decrypt must fail closed on wrong key, wrong
AAD, or tampered ciphertext.

## Database Direction

Add a user crypto table for:

- KDF algorithm and params,
- encrypted `masterKey`,
- encrypted recovery material,
- encrypted `vaultKey`,
- public key,
- encrypted private key,
- crypto schema version.

Add encrypted payload storage for:

- bookmarks,
- bookmark links/text/assets,
- tags,
- lists,
- highlights,
- reading progress anchors where needed.

Legacy plaintext columns can stay during development and migration, but
vault-mode writes must leave private plaintext columns null or non-sensitive.

Blind indexes may be added for exact URL/tag/list matching. They must be keyed
client-side and never computed by the server from plaintext.

## Server Feature Gates

Vault-mode records must not enter plaintext-dependent server paths:

- crawler and recrawler,
- search indexing,
- OpenAI tagging and summarization,
- asset text extraction,
- RSS generation,
- public list rendering,
- plaintext webhooks,
- admin bookmark debugger previews,
- event logs containing URL/title/domain/content.

These paths should either no-op, return an explicit vault-mode unsupported
error, or emit metadata-only events.

## Client Flows

Setup:

1. Authenticate with the server normally.
2. Generate `masterKey`, `vaultKey`, recovery key, and sharing keypair.
3. Derive `keyEncryptionKey` from passphrase.
4. Upload only encrypted key material and public key.
5. Show and confirm recovery key.

Unlock:

1. Fetch encrypted key material and KDF params.
2. Derive `keyEncryptionKey` locally.
3. Decrypt `masterKey`, then `vaultKey` and private key.
4. Keep decrypted keys in memory for MVP.

Create/update bookmark:

1. Build private payload locally.
2. Generate or reuse `bookmarkKey`.
3. Encrypt payload with authenticated envelope.
4. Send encrypted payload plus allowed operational metadata.

Read bookmark:

1. Fetch encrypted payloads.
2. Decrypt in the browser after unlock.
3. Never render stale decrypted data after lock.

## Implementation Order

1. Shared crypto primitives and test vectors (#2).
2. Database/API encrypted envelope support (#4).
3. Web vault setup/unlock and key lifecycle (#3).
4. Encrypted bookmark CRUD MVP (#5).
5. Server worker/API privacy hardening (#7).
6. Client-side search/import/export and migration (#6).
7. Encrypted assets (#9).
8. Recovery, rotation, and multi-device hardening (#8).
9. Encrypted sharing and public surface policy (#10).
10. Security checklist and release gate (#11).

## Open Decisions

- Final JS crypto library: libsodium-compatible APIs match Ente best, but the
  chosen package must work in the web app and have a credible path for mobile,
  extension, and CLI.
- Whether list keys should be introduced in the MVP or delayed until sharing.
  Per-bookmark keys wrapped by `vaultKey` are simpler; list keys make future
  sharing closer to Ente collection keys.
- Which exact operational fields remain plaintext by default.
- Whether blind indexes are needed for MVP or can wait until duplicate URL and
  exact filter UX are implemented.
- How much existing plaintext data migration is needed before personal use.
