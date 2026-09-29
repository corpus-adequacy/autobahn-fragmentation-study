# Autobahn fragmentation comparison — review version 2 (2026-09-29)

Public review version. The raw reports remain private. This version must not be overwritten after sharing; corrections require a new version.

## 1. Bounded result

The accompanying table was extracted from the 30 retained original case reports. Each cell below was identical in all three repetitions. Tuples are `(behavior, behaviorClose)`.

| Server variant | Case 5.6 | Case 5.18 |
|---|---|---|
| R — unmodified websockets reference | OK / OK | OK / OK |
| B — our baseline | OK / OK | OK / OK |
| I — inert control | OK / OK | OK / OK |
| P — suppress interleaved Pong, positive control | FAILED / OK | OK / OK |
| M — ignore the unexpected second TEXT | OK / OK | OK / FAILED |

Autobahn's stock cases were unchanged, at suite revision `b8a5120d905e30470e4475785c48e4cedc35f6cd`, using the config/CLI route in fuzzingclient mode. The deliberate changes were in our own server, not in websockets. We used the two cases to examine this bounded setup; they do not represent the full suite.

The M change alters the declared outcome of stock 5.18 in the closing-handshake field. This is not evidence of a checking gap: we did not establish RFC-invalid server wire behavior accompanied by a green stock verdict. RFC attribution remains inconclusive. Stock 5.6 did not exercise this particular M witness. There is no aggregate score or conformance claim.

## 2. Testsuite description for review

### 2.1 Cases and intended faults

The stock sources are fixed at `b8a5120d905e30470e4475785c48e4cedc35f6cd`:

- [Case 5.6](https://github.com/crossbario/autobahn-testsuite/blob/b8a5120d905e30470e4475785c48e4cedc35f6cd/autobahntestsuite/autobahntestsuite/case/case5_6.py): TEXT (FIN=0), PING (FIN=1), CONT (FIN=1). The case expects Pong followed by the complete echoed text. Our P control suppresses Pong while fragmentation is active, not all Pongs.
- [Case 5.18](https://github.com/crossbario/autobahn-testsuite/blob/b8a5120d905e30470e4475785c48e4cedc35f6cd/autobahntestsuite/autobahntestsuite/case/case5_18.py): TEXT (FIN=0), then TEXT (FIN=1) instead of CONT. Our M variant ignores the unexpected second TEXT rather than closing at that point. Stock 5.18 does not send our separate continuation challenge.

P targets handling control frames during fragmentation and replying to Ping ([RFC 6455 §5.4](https://www.rfc-editor.org/rfc/rfc6455.html#section-5.4), [§5.5.2](https://www.rfc-editor.org/rfc/rfc6455.html#section-5.5.2); the Ping reply requirement has a received-Close exception). M targets the continuation-opcode rule in §5.4: subsequent fragments use opcode 0. TEXT is a known opcode used in an invalid fragmentation sequence here; this is not the unknown-opcode condition in §5.2. The pinned stock case 5.18 expects immediate connection failure. We distinguish that case expectation from a claim that §6.2 explicitly states an unexpected-opcode failure rule. These are the intended normative targets, not a claim that every changed verdict establishes an observed RFC violation. RFC attribution for M/5.18 remains inconclusive in the retained interpretation.

### 2.2 Exact configuration

The suite ran in `fuzzingclient` mode through its config/CLI interface. These are the two complete retained per-attempt spec variants; the corresponding JSON files preserve the exact bytes. `spec-bindings.json` maps all 30 attempts to their spec and original report digests. All 15 attempts of each case use the same spec bytes.

**Case 5.18**, SHA-256 `51d84d0caf1a3c51f8b31f75a5d51333296c7589ba10b893f247d25b2b67c83c`; [exact file](spec-5.18.json):

```json
{
  "cases": [
    "5.18"
  ],
  "exclude-agent-cases": {},
  "exclude-cases": [],
  "options": {
    "closeHandshakeTimeout": 1,
    "failByDrop": false,
    "openHandshakeTimeout": 5,
    "serverConnectionDropTimeout": 1
  },
  "outdir": "/evidence/reports",
  "servers": [
    {
      "agent": "owned-pilot",
      "url": "ws://127.0.0.1:19000"
    }
  ]
}
```

**Case 5.6**, SHA-256 `6a08af812c99417dbadbd6298d5d5c0d577684ac55c3745dde3c4fa31a216d3e`; [exact file](spec-5.6.json):

```json
{
  "cases": [
    "5.6"
  ],
  "exclude-agent-cases": {},
  "exclude-cases": [],
  "options": {
    "closeHandshakeTimeout": 1,
    "failByDrop": false,
    "openHandshakeTimeout": 5,
    "serverConnectionDropTimeout": 1
  },
  "outdir": "/evidence/reports",
  "servers": [
    {
      "agent": "owned-pilot",
      "url": "ws://127.0.0.1:19000"
    }
  ]
}
```

`failByDrop: false` is the study setting. The [discussion guidance](https://github.com/crossbario/autobahn-testsuite/discussions/160#discussioncomment-18521192) distinguishes this strict check from the production preference for `true`. A drop versus a Close frame with code 1002 is not, by itself, evidence of a MUST violation; see [RFC 6455 §7.1.7](https://www.rfc-editor.org/rfc/rfc6455.html#section-7.1.7). We did not run a comparison with `failByDrop: true`.

### 2.3 Verdict and Close fields

[case-results.csv](case-results.csv) contains all 30 executions, including `(behavior, behaviorClose)` and the original `localCloseCode`, `localCloseReason`, `remoteCloseCode`, and `remoteCloseReason`, plus `closedByMe`, `wasCloseHandshakeTimeout`, and `resultClose`. Local and remote are from the suite endpoint's perspective. These are recorded suite fields, not independently verified wire observations. They are copied directly from the original case reports, without using the recovered interpretation to fill any field.

[case-results.json](case-results.json) preserves JSON null values and types; CSV uses literal `null` for JSON null and an empty cell for an empty string. Boolean values in CSV use `true` and `false`. Each row includes its original report SHA-256. All original report bytes were rehashed during preparation of this version.

In all three P/5.6 reports, all four Close code/reason fields are `null`; `closedByMe` is `true`, `wasCloseHandshakeTimeout` is `false`, and `resultClose` is "Connection was properly closed". In all three M/5.18 reports, both codes are `1001`, both reasons are "Going Away", `closedByMe` is `true`, `wasCloseHandshakeTimeout` is `false`, and `resultClose` is "The connection was failed by the wrong endpoint". These are suite-recorded observations, not an independent wire reconstruction. We do not describe the M result as a recorded Close-handshake timeout or as correct server handling of the protocol error. `resultClose` is the suite diagnostic, not a received Close reason.

Thirty executions reached complete terminal records; that is not a claim of zero inconclusive findings. The original end-to-end command failed in postprocessing (§3), and RFC attribution remains inconclusive. A changed closing-handshake verdict alone does not establish forbidden server octets. No RFC-invalid server wire behavior with a green stock verdict was established, and no aggregate adequacy score is reported.

## 3. Original execution and recovered interpretation

All 30 planned case executions reached terminal complete records. The original owner command then failed during report assembly: predecessor records were absent from the blob store used by the interpreter. We preserved the original packet, including the failed postprocessing record. A separate recovery copy supplied two original predecessor blobs and one explicitly derived historical decision blob whose digest matches the original admission reference and supported offline interpretation without rerunning a case.

A subsequent read-only review by an AI code-review agent (Muse) recomputed the recovered case classifications from retained records and accepted that derived interpretation. Reviewer authentication and organisational independence were not established. It did not accept the original command as a successful end-to-end run, and it did not independently establish live container behavior. The exact report hashes are in `case-results.csv`; the recovery and interpretation digests are in `pins-and-recovery.json`. Raw supporting records are retained and are not included in this summary bundle.

## 4. Limitations to retain with the results

- The reference was admitted on licence and provenance, rather than after agreement with both projects as our original proposal said. It was unmodified and is not the subject of a defect claim.
- Docker-host configuration fields were not retained for the run. Resource Saver was enabled at a later inspection; that observation does not establish its setting during the run.
- The full intended subject source tree was not checked during execution. Later source checks do not retroactively supply that missing run-time check.
- For the separately recovered interpretation, the retained records are internally consistent and complete relative to the predefined 30-row plan. They are not independently retained evidence that this is the only or latest execution history. The historical stock ledger was v0, without the later checkpoints.
- Retained wire ordering is not a causal claim. The separate continuation probe is our own case, not stock 5.18, and is excluded from this table and denominator.

The result is a bounded comparison of declared stock outcomes on retained evidence. This summary does not release the underlying raw records. A review of the testsuite description must not be presented as validation of the measurements, endorsement, or a review of data the reviewer has not inspected.

Prepared with AI assistance; the table was checked against the retained original reports.

## 5. Version history

Review version 2 corrects the publication-status line and section heading, adds three suite diagnostic fields for every execution, and corrects the normative reference in §2.1. Version 1 remains available at commit `76843a2624ed4f5c833b400fbee106250f07a338`. No case was rerun and no declared verdict changed.
