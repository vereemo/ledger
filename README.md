# Vereemo Public Proof Ledger

This repository is the **public, append-only fingerprint trail** of every age-verification, identity, and compliance proof issued by the Vereemo platform.

It contains **zero personal data**. It contains only cryptographic hashes and metadata that prove a verification happened, when it happened, and that it has not been altered since.

---

## What lives here

Every UTC day, a single file is committed:

```
bundles/YYYY-MM-DD.jsonl
```

Each line is one proof artifact in JSONL format:

```json
{
  "proof_id": "uuid",
  "proof_type": "age_verification | identity_check | custody_handoff | ...",
  "verifier_id": "stripe_identity | korona_pos | ...",
  "artifact_hash": "sha256...",
  "chain_previous_hash": "sha256...",
  "signature": "hmac-sha256...",
  "signed_at": "2026-05-13T14:22:01Z",
  "proof_summary": {
    "outcome": "verified",
    "age_band": "21+",
    "provider": "stripe_identity"
  }
}
```

The `proof_summary` field is whitelisted server-side. Names, IDs, dates of birth, document numbers, addresses, photos, and phone numbers **never leave Supabase**.

---

## The four-layer permanence stack

A single proof, once committed, becomes irrefutable through four independent systems. To forge a Vereemo proof, an attacker would have to compromise **all four** simultaneously, retroactively, and in coordination — which is computationally and operationally infeasible.

| Layer | What it provides | Operator | Cost |
|---|---|---|---|
| 1. Supabase Postgres | Live database + HMAC signature + hash chain | Vereemo | Paid |
| 2. GitHub (this repo) | Public git history, immutable commits | GitHub | Free |
| 3. Software Heritage | UNESCO/INRIA archive mirror | UNESCO + INRIA | Free, permanent |
| 4. Bitcoin via OpenTimestamps | Proof-of-existence anchored in the world's most expensive blockchain | Decentralized | Free |

### How Bitcoin fits in

Bitcoin does **not** store the proofs themselves. Each daily bundle is hashed into a **Merkle tree**. The single 32-byte root of that tree is submitted to OpenTimestamps, which batches it into a Bitcoin transaction.

```
proof_1 ──┐
proof_2 ──┼─ SHA-256 ─┐
proof_3 ──┘           ├─ SHA-256 ─┐
proof_4 ─── SHA-256 ──┘           ├── Merkle root ──→ Bitcoin tx
proof_5 ───────────────── SHA-256 ┘
```

If a single byte in any proof is changed, the root no longer matches the timestamped hash. The Bitcoin block — mined by a global, adversarial network of miners — becomes the legal evidence that the proof existed in its current form before that block was mined.

Each bundle's commit message includes the Merkle root, e.g.:

```
chore(ledger): 2026-05-13 bundle (11 artifacts) merkle:80824c3c187c
```

You can verify any proof's permanence record at:
```
https://vereemo.com/verify?proof_id=<uuid>
```

---

## Why this exists

Vereemo issues **legal proof artifacts** that dispensaries, hotels, pharmacies, and other regulated venues rely on for compliance. A regulator, auditor, or court asking "did this verification really happen on this date?" must be answerable not only by Vereemo, but by independent third parties.

This ledger is that third party. It is:

- **Public** — anyone can clone, audit, or mirror it
- **Append-only** — the git history makes silent edits impossible
- **Independently archived** — Software Heritage will keep it even if Vereemo and GitHub both disappear
- **Bitcoin-anchored** — the cryptographic commitment is preserved in a global blockchain that no single entity controls

---

## What you will NOT find here

- No names, emails, phone numbers, or addresses
- No identity document numbers or photos
- No dates of birth or biometric data
- No GPS coordinates (only privacy-preserving 7-character geohashes are used internally)
- No customer-recoverable PII of any kind

Vereemo's privacy model enforces 90-day PII retention inside Supabase, with field-level AES-256 encryption. The hashes published here are derived from data that is itself nullified after 90 days — the proof of verification persists, the personal data does not.

---

## How to verify a bundle

```bash
1. Clone the ledger
git clone https://github.com/vereemo/ledger
cd ledger

2. Pick a bundle and recompute its Merkle root
node -e '
  const fs = require("fs"), { createHash } = require("crypto");
  const lines = fs.readFileSync("bundles/2026-05-13.jsonl", "utf8")
    .trim().split("\n").map(JSON.parse);
  const sha = (s) => createHash("sha256").update(s).digest("hex");
  let level = lines.map((l) => l.artifact_hash);
  while (level.length > 1) {
    const next = [];
    for (let i = 0; i < level.length; i += 2)
      next.push(sha(level[i] + (level[i+1] ?? level[i])));
    level = next;
  }
  console.log("Recomputed Merkle root:", level[0]);
'

3. Compare against the commit message of that day
git log --oneline -- bundles/2026-05-13.jsonl
```

If the recomputed root matches the one in the commit message — and that commit is older than the OpenTimestamps Bitcoin block referenced in `permanence.ots_proof` — the bundle is provably authentic.

---

## License

MIT — the data structure and verification scheme are open. The proof artifacts themselves are produced by the Vereemo platform under the platform's terms of service.

---

**Operator:** Vereemo — `info@vereemo.com`  
**Source:** [vereemo.com](https://vereemo.com)
