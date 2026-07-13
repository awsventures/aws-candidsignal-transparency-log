# aws-cbd transparency log

Public mirror of Signed Tree Heads (STHs) for AWS Confidential-by-Design's
Merkle transparency log (RFC 6962/9162). This repo exists so an outside
auditor has a place to fetch STHs from that **we (the operator) cannot
silently rewrite**: git history is the append-only record, and a force-push
to alter a past commit is visible to anyone who already holds a clone or has
run `git log` against this repo before.

## What this proves, and what it doesn't

This log does **not** know who responded to a survey, or what they answered.
What it proves is *counting integrity*: every issued token and every redeemed
(spent) token is appended to a Merkle tree, and each published STH is an
Ed25519-signed commitment to the tree's current root. An auditor who has an
earlier STH and a later one can prove the later tree is a strict
append-only extension of the earlier one (a consistency proof) — so entries
cannot be quietly deleted or reordered after the fact. A respondent who holds
their own receipt can prove their entry is included in a specific published
tree (an inclusion proof) without revealing which entry is theirs to anyone
else.

The cryptographic design (trust model, what's unconditional vs.
conditional-but-auditable) is documented in the main repo:
[`docs/04-implementation-plan/adr/`](https://github.com/awsventures/aws-confidential-by-design/tree/main/docs/04-implementation-plan/adr),
specifically the integrity-anchor and trust-model-and-boundary ADRs.

## Files in this repo

- `latest.json` — the most recently published Signed Tree Head (STH).
- `sth-<size>.json` — every STH ever published, kept permanently (history,
  not just the latest — this is what makes append-only checkable).
- `logkey.spki.hex` — the log's Ed25519 public key (SPKI, hex-encoded), for
  verifying STH signatures.

Each STH is JSON: `{ size, rootHash, at, signature }` — `size` is the tree's
leaf count, `rootHash` the Merkle root over all entries, `at` a coarse
(hour-granularity) timestamp, `signature` the Ed25519 signature over the
canonical encoding of the rest.

## How to verify (no trust in us required)

The verification tool is the public, infrastructure-free audit CLI that ships
in the main repo — it reads only plain JSON files, makes no network calls,
and trusts nothing but the log public key you already have (`logkey.spki.hex`
in this repo).

Clone both repos, then from the main repo:

```sh
git clone https://github.com/awsventures/aws-confidential-by-design.git
git clone https://github.com/awsventures/aws-cbd-transparency-log.git
cd aws-confidential-by-design/packages/anonymity-core

# 1. Verify the published tree head is genuinely signed by the log key.
node --experimental-strip-types --disable-warning=ExperimentalWarning \
  audit/audit.ts verify-sth \
  ../../../aws-cbd-transparency-log/latest.json \
  ../../../aws-cbd-transparency-log/logkey.spki.hex

# 2. Verify two STHs are consistent — the later tree is a strict append-only
#    extension of the earlier one (nothing was deleted or reordered).
node --experimental-strip-types --disable-warning=ExperimentalWarning \
  audit/audit.ts verify-consistency \
  ../../../aws-cbd-transparency-log/sth-<older-size>.json \
  ../../../aws-cbd-transparency-log/sth-<newer-size>.json

# 3. A respondent verifies their OWN receipt is included in a published tree
#    (needs their entry + inclusion proof, obtained from the deployment they
#    responded through, e.g. the mock-vendor sandbox's /audit/locate endpoint
#    — see sandbox/README.md in the main repo for the exact file-format
#    contract of entry-<i>.json / proof-<i>.json / locator.json).
node --experimental-strip-types --disable-warning=ExperimentalWarning \
  audit/audit.ts verify-inclusion \
  <entry-file> <index> \
  ../../../aws-cbd-transparency-log/latest.json \
  <proof-file>

# 4. Check that redeemed counts never exceed issued counts for a survey,
#    over the full ndjson log export (rejects any corrupted line).
node --experimental-strip-types --disable-warning=ExperimentalWarning \
  audit/audit.ts check-counters <entries.ndjson>
```

Exit code `0` = PASS, `1` = FAIL (a real cryptographic failure — tampering,
inclusion mismatch, broken consistency), `2` = malformed input. All four
commands print a human-readable PASS/FAIL line.

## Publication

STHs land here via `packages/services/src/git-sth-target.ts` in the main
repo (`publishSthToGitRepo`): a small wrapper that syncs a local working copy
of this repo, writes the new STH files (reusing the same `publishSth` that
writes every other target), commits, and pushes. It shells out to `git`
rather than depending on a git library, so this repo's history is exactly
what a plain `git clone` + `git log` would show you — no special tooling
needed to audit the audit trail itself.

## Initial seed

The first STH in this repo (`sth-1.json` / `latest.json`) was published from
a single real respondent completing the full issue → redeem → submit flow in
the [mock-vendor sandbox](https://github.com/awsventures/aws-confidential-by-design/tree/main/sandbox)
demo, not synthetic test fixtures — the same cryptographic path a real survey
integration uses.
