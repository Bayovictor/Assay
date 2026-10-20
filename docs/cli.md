# The CLI

`assay` is one binary with five subcommands. All of them print human-readable
JSON by default; every subcommand that produces structured output also has a
`-raw` form that prints tab-separated values, for scripts and CI.

```
usage:
  assay scan CODE-ISSUER          classify one asset and print the report as JSON
  assay attestation CODE-ISSUER   print the on-chain attest() arguments for one asset
  assay verify [-hash HEX] [-raw] [PREIMAGE]
                                  check a canonical preimage against an evidence_hash
  assay history [-guarantee] [-raw] CODE-ISSUER
                                  print the asset's observation history
  assay serve [-addr] [-history PATH]
                                  serve the HTTP API and UI
```

Commands that scan also accept:

- `-asset-lists URL[,URL...]` — SEP-0042 Stellar Asset Lists to consume, given
  as a comma-separated list or repeated. **No list is used by default**: nothing
  is hardcoded as authoritative, and no report changes unless you opt in. See
  [asset-lists.md](asset-lists.md).

## Asset identifiers

Every command that takes an asset accepts the same three spellings of it:

| Spelling | Example |
| --- | --- |
| `CODE-ISSUER` (canonical) | `USDC-GA5Z…` |
| `CODE:ISSUER`, as most Stellar tools print it | `USDC:GA5Z…` |
| a stellar.expert asset URL, paste-able from the browser | `https://stellar.expert/explorer/public/asset/USDC-GA5Z…` |

They are normalised on input, and the canonical `CODE-ISSUER` form is what
every output — report, evidence lines, preimage — prints, so the bytes
`evidence_hash` commits to are identical whichever form you pasted. The URL's
`<network>` segment is not checked: which ledger the scan reads is settled by
the scanner's network binding (#41), not by the address bar.

## assay scan

Scans one asset and prints the full report: the findings, the reasoning behind
each, and the evidence every conclusion is attributed to.

```
./assay scan CODE-ISSUER              # the findings and reasoning
```

The network's native asset is not scannable: it has no issuer account, so
there are no authorization flags to read. `assay scan native` (or `XLM`, any
case) recognises the input and refuses it with an explicit explanation rather
than a parse error, prints nothing to stdout, and exits 2:

```
assay: native asset has no issuer and no authorization flags: "XLM" cannot be scanned or attested
```

Every command that takes an asset — `scan`, `attestation`, `history`,
`verify` — refuses it the same way, so the native asset can never be scanned
or attested from the CLI. A credit asset whose code merely happens to be XLM
(`XLM-G…`) is scanned as the ordinary asset it is.

## assay attestation

Derives the arguments of an on-chain `attest()` call from a live scan. It
deliberately does not submit anything: signing belongs to whoever holds the
attester key, and keeping derivation separate from submission means the
numbers going on-chain can be inspected before a key ever touches them.

```
./assay attestation CODE-ISSUER       # severity, flags, evidence_hash
./assay attestation -preimage CODE-ISSUER   # + the bytes the hash commits to
./assay attestation -raw CODE-ISSUER  # just `SEVERITY<TAB>FLAGS<TAB>HASH`
```

`-preimage-version vN` renders the report under a historical encoding instead
of the newest one it binds, and accepts `v1`, `v2`, `v3` or the full
`assay-evidence-vN` line:

```
./assay attestation -preimage -preimage-version v1 CODE-ISSUER
```

This is how an attestation already on chain is reproduced: the recorded
`evidence_hash` was hashed over the bytes of its recorded version, so a
verifier must re-render those bytes rather than compare against whatever the
current scanner emits. The version the report itself binds is unchanged; only
the rendering is. An unknown version is refused before anything is printed.

With `-raw` and no `-preimage-version`, the output is unchanged: three fields,
`SEVERITY<TAB>FLAGS<TAB>HASH`. With `-preimage-version`, a fourth field is
appended — the hash the report renders under the version the current binary
would pick — so a caller can tell evidence drift from metadata drift:

```
SEVERITY<TAB>FLAGS<TAB>HASH(version)<TAB>HASH(current)
```

`scripts/reproducibility.sh` is the reference consumer: when the first hash
does not match the recorded one but the second does, the evidence is unchanged
and it reports `ENCODING` rather than a misleading `MISMATCH`.

## assay verify

Checks a canonical preimage against an `evidence_hash`. It is the third-party
half of `attestation`: the attester publishes the bytes with
`assay attestation -preimage`, the registry stores the hash, and `verify`
decides whether the two correspond. It replaces the two commands and a human
comparing hex that verification used to be.

It reaches nothing. The preimage names its own asset, and the hash either
matches these bytes or it does not. The preimage is read from a file, or from
stdin when the argument is omitted or `-`.

```
./assay verify -hash 0x688453bd… preimage.txt   # verdict for a published preimage
./assay verify -hash "$(cat digest.txt)" < preimage.txt
./assay attestation -preimage CODE-ISSUER | jq -r .preimage | ./assay verify -hash 0x688453bd…
./assay verify -raw -hash 0x688453bd… preimage.txt   # just the recomputed hash
```

Flags:

- `-hash` — the `evidence_hash` to check against, 64 hex characters, with or
  without a `0x` prefix and in either case. Omitted, the command prints the hash
  the bytes produce, which is the value to compare against what is stored.
- `-asset` — also require the preimage to be for this `CODE-ISSUER`, so a
  correct hash for the wrong asset cannot pass.
- `-raw` — print only the recomputed hash.
- `-quiet` — print nothing and report the verdict through the exit status alone.

Semantics:

- **match** — JSON with `match: true`, exit 0.
- **mismatch** — JSON carrying both `computed` and `claimed`, exit non-zero.
  Those two values are the whole answer, so they are printed before the failure.
- **malformed hash** — refused as its own error rather than reported as a
  mismatch, so a truncated or mistyped hash is not mistaken for a broken
  attestation.
- **unreadable header** — noted on stderr, and the hash is still checked. The
  bytes are what the hash covers, so a cosmetic problem must not hide a real
  mismatch.

An unrecognised version line is a note rather than a failure, but it is not
harmless: the version is inside the hash, so a version this build does not know
reports itself rather than being compared against v1. A v1 preimage binds no
check set and is reported as `unknown`, never as complete — the rule
`docs/contract-interface.md` sets out.

## assay history

Prints an asset's observation history: one line per recorded observation, in
time order, derived from the report's evidence entries. Where `scan` answers
"what is true now", `history` answers "what did we observe, and when" — the
view an auditor needs when an asset's state changed and someone wants to know
whether it was ever different.

Each entry carries the asset, the report's severity at scan time, the
**transition** — what the issuer was observed doing, reduced to a term
(`malicious`, `listed as`, `blocked`, `unverified`, `not retrievable`, …) — the
observation's reason verbatim, and the time it was retrieved.

An observation that does not reduce to a known transition prints as
`unknown`. The history view deliberately does not re-derive the verdict; it
reports what each observation said.

```
./assay history CODE-ISSUER           # the observations as JSON
./assay history -raw CODE-ISSUER      # `ASSET<TAB>SEVERITY<TAB>TRANSITION<TAB>REASON`
```

Flags:

- `-raw` — print the history tab-separated instead of as JSON, for scripting.
- `-guarantee` — exit non-zero when the asset has no observations, so a script
  can distinguish "no history" from "history ran fine" without parsing output.

Semantics:

- **valid state** — the observations print in time order.
- **missing state** — a clear message (`no observations for ASSET`), and exit
  0 unless `-guarantee` was given.
- **unknown state** — an observation whose claim matches no known transition
  prints with transition `unknown` rather than being dropped.

## assay serve

Serves the HTTP API and UI. The API exposes the same reports the CLI prints;
see `docs/integrating.md` for what a consumer reads out of it, and
[history.md](history.md) for the observation history endpoint.

```
./assay serve -addr :8080
./assay serve -addr :8080 -history /var/lib/assay/history.jsonl
```

Each successful `GET /api/v1/scan` records an observation, and
`GET /api/v1/history?asset=CODE-ISSUER` returns them in time order with the
derived transitions. `-history` points that store at an append-only JSON Lines
log so it survives a restart; without it, history is in memory and lost when the
process exits. The default retention is 256 observations per asset and the
policy is in [history.md](history.md).

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success. |
| 1 | Generic failure: the command ran but could not deliver a result. |
| 2 | Usage error: the command or its arguments cannot be used as given — an unknown command, a missing or malformed `CODE-ISSUER`, or an identifier naming the native asset, which is never scannable. |
| 3 | Not found: the asset or attestation does not exist on the ledger. |
| 4 | Upstream unavailable: a source the scan depends on could not be reached. |
| 5 | Undetermined: the scan ran, but a source outage or an inconsistency left no verdict to report. |

The native-asset refusal is deliberately a usage code rather than a generic
failure: it is answered from the ledger's own rules before any fetch, so it is
permanent and no retry will change it — a script can distinguish "this
identifier will never scan" from a transient failure without parsing stderr.
The same input is answered by the HTTP API as `400 Bad Request` with the same
message; see [api.md](api.md).
