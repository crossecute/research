# Aave nonce 447 evidence

Chain records were collected on 2026-11-06. The verdicts were generated on 2026-11-07 with Forge Attest v1.4.0 (`9e76385ac2f96f171ceb9391cf9e8298e2d408cf`), Foundry v1.8.1 and `--require-live --json`. All three runs passed.

| File | Origin and meaning |
| --- | --- |
| `safe-service-nonce447.json` | Safe Transaction Service record for Risk Council `0x47c7…0fA8`, nonce 447, safeTxHash `0x0ea9a8fe…8e4`. Executed and successful at block 26,337,526. Both confirmations are approved hashes. |
| `safe-service-child-nonce18.json` | Service record for child Safe `0x2B99…B1cA` (2 of 5), nonce 18, safeTxHash `0xf7ebe808…52e5`. Plain `approveHash(parentHash)`. |
| `safe-service-child-nonce205.json` | Service record for child Safe `0xb291…Afef` (1 of 3), nonce 205, safeTxHash `0xcce0f737…07cb`. `approveHash(parentHash)` followed by the parent transaction. |
| `execution-receipt.json` | Receipt for Ethereum transaction `0x80169ec553baa63195fefd7ac61e6329879916284944f67fec7f82fd737dd473`. Status is `0x1`. The Risk Council's `ExecutionSuccess` log carries the nonce 447 safeTxHash. Read from `eth.drpc.org`. |
| `fork-block.json` | Ethereum block 26,337,525, the fork block of the capture claim, read from `eth.drpc.org`. |
| [`ci-run.json`](ci-run.json) | `gh run view` output for run 42303115084 on the demo's v1.4.0 branch at `e6ce75bb1ec25abe5588d6e36ce41184ed38239f`. All four jobs passed. |
| [`attestation-plain.json`](attestation-plain.json) | Verdict for [`attest.toml`](../attest.toml), covering the parent and the plain approval at child nonce 18. |
| [`attestation-preimage-suffix.json`](attestation-preimage-suffix.json) | Verdict for [`attest.child-a.toml`](../attest.child-a.toml), covering the parent and the approval with the parent transaction suffix at child nonce 205. |
| [`attestation-capture.json`](attestation-capture.json) | Verdict for [`attest.capture.toml`](../attest.capture.toml), covering the parent through the intercepted `safe-utils` request. The fork RPC was `eth.drpc.org`. |

The three claim files are copies from the demo commit cited above. The verdicts record the earlier producer commits that the claims tell Forge Attest to run.

Public test private key `1` satisfies the capture script's local signing step. The signature in the captured request is not an owner approval, and nothing was submitted to the Safe service.
