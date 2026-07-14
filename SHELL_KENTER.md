# SHELL_KENTER.md — Kenter on the Keycard Shell (agent spec)

**Audience:** the agent working in this repo (`keycard-shell`, STM32H5 / FreeRTOS / C firmware).
**Status:** implementation spec (2026-07-14). Grounded in a read of both forks —
this firmware and the `status-keycard` applet (the Ed25519 fork).
**Goal:** make the Keycard Shell an **air-gapped Kenter signer + verifier** for
Kenter's **spend mode**.

---

## Repos & source references

| Repo | What | Link |
|---|---|---|
| **keycard-shell** (this repo) | STM32H5 device firmware | <https://github.com/inviti8/keycard-shell> (branch `master`) |
| **status-keycard** | The JavaCard applet — on-card Ed25519 (DONE) | <https://github.com/inviti8/status-keycard/tree/feat/stellar-ed25519> |
| **kenter** *(private)* | The protocol: crypto, mint contract, spend mode | <https://github.com/inviti8/kenter> (branch `main`) |

**Kenter crypto — the source of truth this spec pins** (verify byte layouts against it):
- Spend-mode primitives: [`crates/kenter-crypto/src/spend.rs`](https://github.com/inviti8/kenter/blob/main/crates/kenter-crypto/src/spend.rs)
- Transfer protocol + client verification gates: [`SPEND_MODE.md`](https://github.com/inviti8/kenter/blob/main/SPEND_MODE.md)
- Design rationale + locked decisions: [`NULLIFICATION.md`](https://github.com/inviti8/kenter/blob/main/NULLIFICATION.md)
- Live testnet contract addresses: [`docs/DEPLOY.md`](https://github.com/inviti8/kenter/blob/main/docs/DEPLOY.md)
- Mobile/air-gap product context: [`PIVOT.md`](https://github.com/inviti8/kenter/blob/main/PIVOT.md) (§P5 custody)

**Card applet (status-keycard, `feat/stellar-ed25519`)** — the Ed25519 the Shell drives:
- [`Ed25519.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/main/java/im/status/keycard/Ed25519.java) — `signHash` / `publicKey` / SLIP-0010
- [`KeycardApplet.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/main/java/im/status/keycard/KeycardApplet.java) — `INS_SIGN`, `INS_EXPORT_KEY`, P1/P2 constants
- Applet harness: [`Ed25519ProbeTest.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/test/java/im/status/keycard/Ed25519ProbeTest.java), [`StellarM4Test.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/test/java/im/status/keycard/StellarM4Test.java) (jcardsim)

**This firmware — files this spec touches** (relative):
[`app/crypto/`](./app/crypto/) · [`app/core/core.c`](./app/core/core.c) ·
[`app/core/core_eth.c`](./app/core/core_eth.c) ·
[`app/keycard/keycard_cmdset.c`](./app/keycard/keycard_cmdset.c) /
[`.h`](./app/keycard/keycard_cmdset.h) · [`app/ur/`](./app/ur/)

**Live mint contract (testnet):**
[`CB6QDGPL…RJAK`](https://stellar.expert/explorer/testnet/contract/CB6QDGPLL7JY76TS5PH73BYSCTGPHODA4NDDUI7VIFALRYPEJ6JLRJAK)

---

## 0. TL;DR

Make this Shell an **air-gapped Kenter signer + verifier** for **spend mode**. It
does two jobs, both offline:
1. **Verify** a received bearer token's structure (Ed25519 chain sigs) — no network.
2. **Sign** ownership operations with the on-card Ed25519 **owner key**: transfer
   (`spend`) and redeem authorizations.

It does **not** touch the ledger. A companion **phone** broadcasts the signed
`spend` and runs the online gates (`get_token` polling). QR (animated UR) is the
only channel between them.

The card fork (`status-keycard`) already provides the Ed25519 primitives. This
firmware needs Ed25519 **verify** added, the card's Ed25519 **sign** wired, and a
**`core_kenter.c`** flow. One Kenter-side change is required: sign a **32-byte
digest** (§3.1).

---

## 1. The Shell's role (and what it is NOT)

Spend mode (Realization A) prevents the payer from redeeming after paying, via an
on-ledger **owner pointer** the payer moves and redemption checks (see
[`SPEND_MODE.md`](https://github.com/inviti8/kenter/blob/main/SPEND_MODE.md)).
Mapping to an **air-gapped** device:

| Job | Who |
|---|---|
| Custody the owner key; sign `spend` / redeem authorizations | **Shell** (card) |
| Verify a received token offline (chain integrity) — gate **G1** | **Shell** |
| Broadcast the `spend` tx; poll `get_token` for **G2/G3** (Active + owned-by-me) | **phone** (online) |
| Transport (token bytes, spend request, signed result) | **QR / animated UR** |

The Shell **cannot** verify ledger state itself (air-gapped) — that's the phone's
job, or a signed "visual oracle" (a trusted server rendering `get_token` results as
QR the Shell scans; ≈ a phone trusting its RPC). The Shell is a **signer**, never
an online wallet. It stores keys, not tokens.

---

## 2. The two forks and the seam

- **`status-keycard` (card applet, DONE):**
  [`Ed25519.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/main/java/im/status/keycard/Ed25519.java)
  — pure Ed25519 (RFC 8032) `signHash(secret, hash32) → (pubkey32, sig64)`,
  `publicKey(secret)`, and **SLIP-0010 ed25519** child derivation (hardened only,
  `m/44'/148'/n'`).
  [`KeycardApplet.java`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/main/java/im/status/keycard/KeycardApplet.java):
  `INS_SIGN=0xC0` with `SIGN_P1_DERIVE=0x01`, algo selected by P2;
  `INS_EXPORT_KEY=0xC2` with `EXPORT_KEY_P2_ED25519_PUBLIC=0x03`.
  **The card signs a 32-byte hash and derives keys by path.**
- **`keycard-shell` (this firmware, TO BUILD):** today a BTC/ETH air-gapped signer.
  [`app/crypto`](./app/crypto/) is trezor-crypto **without Ed25519**; cores are
  [`core_btc.c`](./app/core/core_btc.c) / [`core_eth.c`](./app/core/core_eth.c);
  [`keycard_cmd_sign(kc, algo, path, path_len, hash)`](./app/keycard/keycard_cmdset.c)
  exists but `KEYCARD_SIGN_EDDSA_ED25519` is `/* unsupported */` in
  [`keycard_cmdset.h`](./app/keycard/keycard_cmdset.h).

**The seam:** the firmware calls the card with `(algo, path, hash)` and gets back
`(pubkey, sig)`. Wiring the `EDDSA_ED25519` algo through is the whole card link.

---

## 3. The Kenter crypto contract (pin exactly)

All bytes must match
[`kenter-crypto/src/spend.rs`](https://github.com/inviti8/kenter/blob/main/crates/kenter-crypto/src/spend.rs)
and the mint contract, or signatures won't verify on-chain. Cross-check against
that crate's test vectors.

- **Owner commitment:** `commitment = SHA-256("hvym_v1:spend:owner" || owner_pubkey_32)`.
  The Shell exports the Ed25519 owner pubkey from the card, computes this, and the
  32-byte commitment is what goes on-ledger. A **fresh owner key per receipt** =
  derive `m/44'/148'/n'` at a new index `n`; store only `n`.
- **Spend authorization message:**
  `"hvym_v1:spend:auth" || ephemeral_key_32 || current_commitment_32 || new_owner_commitment_32`.
- **Redeem authorization message:**
  `"hvym_v1:spend:redeem" || ephemeral_key_32 || owner_commitment_32`.

### 3.1 REQUIRED Kenter-side change: sign a 32-byte digest

The card's `signHash` takes a **32-byte** message. The Kenter messages above are
84–114 bytes, so they cannot be fed to the card directly. Resolution (a Kenter-side
change, pre-mainnet, cheap):

> **Owner authorizations sign `M = SHA-256(message)` (32 bytes), not the raw
> message.** The Shell computes `M`, calls the card `signHash(owner_secret, M)`,
> and the mint `ed25519_verify`s over `M`. Software (`kenter-crypto`) must sign the
> same digest so software- and card-produced signatures are interchangeable.

**Owner-side changes (kenter repo, done before wiring the card):**
`kenter_crypto::spend::{authorize_spend, verify_spend, authorize_redeem,
verify_redeem_owner}` hash the message with SHA-256 before sign/verify; the mint
`spend` + `enforce_owner` `ed25519_verify` over `sha256(message)`; update the
contract test helpers; redeploy. (The *ephemeral*-key redeem signature — Path A's
`domain_tag || ephemeral_key` — is unaffected; that's the composed key, signed by
the combiner in software, not the card.) **This is an open decision for the Kenter
maintainer — coordinate before the card path relies on it.**

The Shell only ever sees `M` (32 bytes) — it never needs the message layout beyond
building it to hash. Keep the layout identical to `spend.rs`.

---

## 4. Card interface the Shell uses (grounded)

- **Derive + export owner pubkey:** `INS_EXPORT_KEY` P1=`DERIVE`(0x01)
  P2=`ED25519_PUBLIC`(0x03), path `m/44'/148'/n'` → 32-byte Ed25519 pubkey. Add an
  export path alongside the existing secp256k1 export.
- **Sign:** `keycard_cmd_sign(kc, KEYCARD_SIGN_EDDSA_ED25519, path, path_len, M32)`.
  Under the hood this is `INS_SIGN` P1=`DERIVE`(0x01), the algo mapped to the
  applet's Ed25519 branch (`Ed25519.signHash`), returning a signature template
  `pubkey(32) || sig(64)` (parse per the applet's `TLV_SIGNATURE_TEMPLATE`).
- **PIN gate:** signing is PIN-gated (existing `VERIFY PIN` flow) — reuse it.
- **Secure channel:** `SecureChannelV2` / [`secure_channel_v2.c`](./app/keycard/secure_channel_v2.c)
  is curve-agnostic; unaffected by the Ed25519 work.

---

## 5. Firmware work items (this repo)

Grounded in the existing structure — follow the eth/btc patterns.

1. **Ed25519 verify in [`app/crypto`](./app/crypto/).** Absent today. Drop in
   trezor-crypto's `ed25519` module (ed25519-donna); `sha2.c` already provides the
   SHA-512 it needs. Needed for **G1** (verify the token's daisy-chain dual
   signatures + container signature offline) and to sanity-check the card's
   returned signature.
2. **Wire the card Ed25519 sign** ([`keycard_cmdset.c`](./app/keycard/keycard_cmdset.c)):
   flip `KEYCARD_SIGN_EDDSA_ED25519` from `/* unsupported */` to a real path — map
   the algo to the applet's Ed25519 P2, send the 32-byte `M`, parse `pubkey || sig`.
3. **Ed25519 pubkey export** — add the `EXPORT_KEY P2=0x03` path to the client.
4. **`app/core/core_kenter.c`** (mirror [`core_eth.c`](./app/core/core_eth.c)). Add
   tx types to [`core_qr_run`](./app/core/core.c)'s dispatch (alongside
   `ETH_SIGN_REQUEST` / `CRYPTO_PSBT`):
   - **`KENTER_RECEIVE`** — derive a fresh owner key (`m/44'/148'/n'`, next `n`),
     export its pubkey, compute the commitment, **display it as a QR** for the payer
     to spend to. Persist `n` in [`app/storage`](./app/storage/).
   - **`KENTER_SPEND_REQUEST`** — scan a UR carrying the token bytes +
     `new_owner_commitment` (+ which owner index is current). **Verify the token
     (G1)**, show amount + recipient (clear-signing), build the spend message,
     `M = SHA-256(message)`, `keycard_cmd_sign(EDDSA, owner_path, M)`, then **emit a
     UR QR** of `{ephemeral_key, current_owner_pubkey, new_owner_commitment,
     signature}` for the phone to submit to `mint.spend(...)`.
   - *(later)* **`KENTER_REDEEM_REQUEST`** — the redeem owner-proof, same shape.
5. **A Kenter UR type.** Define one alongside the existing UR types in
   [`app/ur`](./app/ur/) (`ur_types.h` / registry), or wrap the payload in the
   generic `bytes` UR. Use **animated** UR so a token (~1.5–3 KB) fits across frames.
6. **UI / clear-signing.** Before any card sign, display token value +
   recipient-commitment fingerprint on-screen (mirror the eth EIP-712 clear-sign
   UX). The user approves via keypad; nothing signs silently.

**Not needed:** Stellar tx building, SEP-0005 addresses, strkey — the Shell signs
raw Kenter digests, never Stellar transactions.

---

## 6. The flows on-device (payer = Shell, payee = phone or 2nd Shell)

```
RECEIVE (payee side)              SPEND (payer = Shell)
────────────────────             ─────────────────────
scan/none                         scan UR: token bytes + payee commitment
derive owner key m/…/n'           G1: verify token offline (ed25519)  ← Shell
export pubkey → commitment        show amount + payee fingerprint, PIN
show commitment QR  ───────────►  M = SHA256("spend:auth"||ek||cur||new)
persist n                         card signHash(owner_key, M) → sig
                                  emit UR QR: {ek, owner_pub, new_commit, sig}
                                          │
                                          ▼
                                  phone submits mint.spend(...) ; polls get_token
                                  (G2 Active + G3 owner==payee) → payee "paid"
```

The Shell's responsibility ends at emitting the signed-spend QR. **G2/G3 are the
online side's** (phone or visual oracle) — see
[`SPEND_MODE.md`](https://github.com/inviti8/kenter/blob/main/SPEND_MODE.md) §4 gates.

---

## 7. Verification & test plan

- **Cross-verify the card signature** against `ed25519-dalek` and the mint: a
  card-produced `(pubkey, sig)` over `M` must (a) verify with `ed25519-dalek`, and
  (b) open the on-chain commitment when submitted to `mint.spend`. The
  `status-keycard`
  [`Ed25519ProbeTest`](https://github.com/inviti8/status-keycard/blob/feat/stellar-ed25519/src/test/java/im/status/keycard/Ed25519ProbeTest.java)
  / `StellarM4Test` + jcardsim are the applet harness; extend with a Kenter `M`.
- **SLIP-0010 vectors:** confirm derived owner keys match SEP-0005 / SLIP-0010
  ed25519 test vectors (the applet already claims this).
- **End-to-end live:** phone submits the Shell-signed spend to the live mint
  [`CB6QDGPL…`](https://stellar.expert/explorer/testnet/contract/CB6QDGPLL7JY76TS5PH73BYSCTGPHODA4NDDUI7VIFALRYPEJ6JLRJAK);
  `get_token` shows the owner pointer moved to the payee commitment. This is the
  same transition the desktop `kenter spend-demo` proves in software — the Shell
  replaces the software owner key with the card.
- **Determinism:** `Ed25519.signHash` is deterministic (no on-card RNG); a fixed
  `(seed, M)` must always give the same `sig` — assert it.

---

## 8. Open inputs / decisions (for the Kenter maintainer)

1. **Approve the 32-byte-digest change (§3.1)** in the kenter repo (spend.rs +
   contract + redeploy). Prerequisite for the card to be the owner signer.
2. **UR type**: dedicated `kenter-*` UR type vs. the generic `bytes` UR — record it
   in the applet/firmware registry.
3. **Owner-key index management**: monotonic `n` per receipt is simplest; decide
   whether to also support a reusable "static receive address" (async payments,
   trades unlinkability for reach — [`PIVOT.md`](https://github.com/inviti8/kenter/blob/main/PIVOT.md) §P5).
4. **Redeem on-Shell**: whether the Shell signs the *redeem* owner-proof (payee) or
   only the *spend* (payer). Same card operation; sequencing is UX.
5. **Visual oracle** (optional, later): signed-attestation QR format for air-gapped
   payee verification — a follow-up doc.
