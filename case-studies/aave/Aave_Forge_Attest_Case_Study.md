# Forge Attest for Aave v4 Risk Council operations

## Gap in the public proposal flow

The Aave v4 steward script simulates the proposed updates, can write a state diff, and uses `safe-utils` to sign and post a MultiSend batch to the Safe Transaction Service. The public flow leaves reviewers to verify independently that the full transaction at the intended Safe and nonce matches the source they reviewed. One of the child Safe approvals for Risk Council nonce 447 carries only the parent hash, so its calldata gives signers no parameter values to review. This assessment covers the published script and the recorded approvals. [1, 3, 5]

## What signers trust with Forge Attest

Forge Attest shifts trust in the proposal to the reviewed script, its checked-in claim, and the CI run that verifies them. The script and claim define the expected transaction. CI reruns the pinned script and records the full Safe transaction hash. Before approving, signers can check that the proposal at the intended Safe and nonce matches that hash. Running with `--require-live` makes the Safe Transaction Service comparison mandatory. [3, 4, 6]

Forge Attest derives each child approval from the same parent transaction, so child signers can check both hashes. The demo uses v1.4.0 to reproduce Ethereum nonce 447, executed on 2026-11-03, and verify both child approvals. It also intercepts the demo's `safe-utils` request locally and verifies that it yields the same parent hash. [1, 3, 4, 6]

## How the research found this Safe

The research surveyed public repos where a Foundry script builds, proposes or executes a Safe transaction. GitHub code search returned nothing for Aave. A later scan of the public repos in the `aave`, `aave-dao` and `bgd-labs` organizations found one active repo of this kind, [`aave-dao/aave-v4-risk-stewards`](https://github.com/aave-dao/aave-v4-risk-stewards/blob/082760efc4e9d8b414d668fb80b3ec544b91b605/scripts/RiskStewardsBase.s.sol). The research classes it as style S4, a script that proposes through the Recon-Fuzz `safe-utils` client. [7]

The script's `run(broadcastToSafe, generateDiffReport, skipTimelock)` simulates the steward calls, can write a diff report of protocol state before and after, and can propose one MultiSend batch. On a chain with no registered MultiSend, such as Arc, it only prints calldata. The research notes that this is a small repo and not Aave's main governance flow, which uses per-proposal Foundry scripts for Aave Governance V3 instead. [5, 7]

The research took the Safe from `RISK_COUNCIL()` on the steward: `0x47c71dFEB55Ebaa431Ae3fbF99Ea50e0D3d30fA8` on Ethereum, threshold 2 of 2 owners. It then matched executed transactions on that Safe against the calls the script builds. Nonce 447 matched at the *selectors* level: the same target and selectors, with arguments not checked. The demo raises that to a byte-for-byte match against the executed calldata. [1, 3, 7]

## What nonce 447 does

The transaction is a MultiSendCallOnly delegatecall (Safe version 1.3.0, chain ID 1) with two calls to the Risk Steward `0x6f48d9Cdb8EE6E17c96B2d8Aec128af426A295c1`. Both calls send zero ETH, and the Safe gas and refund fields are zero. [1, 3]

| Order | Call | Effect |
| --- | --- | --- |
| 1 | `updateHubAssetIRs` (772 bytes) | USDC `rateGrowthBeforeOptimal` set to 550 on the Core Hub and 450 on the Prime Hub. Every other field keeps its current value. |
| 2 | `updateHubSpokeCaps` (1,220 bytes) | On the Core Hub: draw cap 45,000 for WETH on the EtherFi spoke, add cap 25,000 for weETH on the Main spoke, draw cap 40,000,000 for USDC and draw cap 20,000,000 for USDT on the Main spoke. Cap values are as encoded. |

The Safe executed the transaction successfully at Ethereum block 26,337,526 (transaction [`0x8016…d473`](https://etherscan.io/tx/0x80169ec553baa63195fefd7ac61e6329879916284944f67fec7f82fd737dd473)). The MultiSend payload is 2,162 bytes and the Safe transaction data is 2,244 bytes. [1, 2]

## What the demo verifies

| Measure | Result |
| --- | --- |
| Coverage | One transaction with two ordered calls: Safe `0x47c7…0fA8`, Ethereum, nonce 447, Safe version 1.3.0. |
| Source of addresses | The Safe, steward, hub configurator, both hubs, both spokes and the four assets come from `aave-address-book` (pinned at `b71495a`). Only the update values are written in the script. |
| Calldata check | A Foundry test compares both rebuilt calls byte for byte with fixtures decoded from the executed transaction. |
| Verification | Raw and canonical digests are pinned. `cast` and a separate Solidity implementation derive the Safe hash independently. The hash is `0x0ea9a8fed74bc267e88a5de190ca88a3c63b5963784f50176bfd0f4fb523f8e4`. |
| Service record | The live check compares the hash with the transaction the Safe Transaction Service holds at nonce 447. It is already executed. |
| Producer path | A second script proposes the same payload through `safe-utils`, pinned at v0.0.22. Forge Attest runs it unchanged on a fork one block before execution, captures the request it would post, and reaches the same canonical digest and hash. |
| CI | All four jobs passed on the demo's Forge Attest v1.4.0 branch: `forge test`, both nested approval configs and the capture config. The capture job requires a live Safe service match. [3, 4] |

The Safe hash covers the target, value, calldata, operation, gas fields, nonce, chain and Safe address. A changed rate, cap, asset, spoke, call order or omitted call would change it. [3, 6]

The capture config uses Forge Attest's `capture = "safe-utils"` mode, which intercepts the JSON request that `safe-utils` would send to the Transaction Service and returns a local response. Forge Attest v1.4.0 accepts the redirect options sent by the demo's `safe-utils` client. The demo uses public test key `1` to sign the intercepted request locally. Aave's own script signs with a hardware wallet. [3, 6]

## The Safe is nested

Both owners of the Risk Council are Safes. Each approves nonce 447 with its own transaction, `approveHash(parentHash)` on the Risk Council, and its owners sign that. The Safe service shows both confirmations on nonce 447 as approved hashes. [1, 3]

| Child Safe | Threshold | Child nonce | Executed (block) | Form of the approval |
| --- | --- | --- | --- | --- |
| `0xb291…Afef` | 1 of 3 | 205 | 26,336,291 | `approveHash(parentHash)` followed by an ABI-encoded copy of the parent transaction (2,788 extra bytes), so its owners can read what they approve. |
| `0x2B99…B1cA` | 2 of 5 | 18 | 26,337,523 | The plain `approveHash(parentHash)` call. |

Forge Attest builds each approval from the parent transaction. The demo pins both child hashes (`0xcce0f737…07cb` and `0xf7ebe808…52e5`) and compares them with the executed child transactions in the Safe service. `attest.toml` checks the plain approval at child nonce 18. `attest.child-a.toml` sets `child_approval = "preimage-suffix"` to rebuild the parent transaction suffix at child nonce 205. Changing that suffix changes the child hash. The demo author could not identify the tool that wrote the original suffix. [3, 6]

Both approvals bind to the parent hash for Risk Council nonce 447.

## Additional verification and operational benefits

| Review step | Existing public script | With Forge Attest |
| --- | --- | --- |
| Intended calls | Simulates the steward calls and can write a diff report of protocol state. | Rebuilds the calls from pinned source and compares the complete transaction. |
| Source to request | The script builds the batch and `safe-utils` posts it. | Captures that request locally and checks raw and canonical digests. |
| Signed identity | `safe-utils` derives the hash and signs it. | `cast` and a separate Solidity implementation derive the complete hash and compare it with the reviewed expectation. |
| Nested approvals | Each child Safe signs a hash for a parent transaction it does not build. | Builds each child approval from the parent, so child signers can check which transaction they approve. |
| Service record | The proposal path submits to the Safe service. | Compares the hash with the transaction the service holds at the claimed Safe and nonce. |
| Review evidence | Source and the executed transaction can be inspected. | A checked-in claim, CI result and derived hashes make the comparison repeatable. |

These checks establish transaction equality. Reviewers still decide whether the risk parameter change is appropriate, and the steward enforces its own bounds on chain. [3]

## Adoption and scope

For a future Risk Council update, the team can check in a claim next to the script and run Forge Attest in CI. The signers of each owner Safe can then run the same claim before approving. Capture mode runs a `safe-utils` proposal script unchanged. The demo runs its own proposal script for this payload, not Aave's, so this study does not show it running against the steward repo's script. [3, 6]

The demo adds no on-chain transaction, calldata field or execution cost. Reviewer time and net savings were not measured.

## Reproduction, evidence and limits

The commands use demo commit `e6ce75b` and Forge Attest v1.4.0 (`9e76385ac2f96f171ceb9391cf9e8298e2d408cf`). Use Foundry v1.8.1 for the capture claim, which checks the Forge version. The CI jobs for the nested claims use the action's default Foundry v1.7.1. [3, 4, 6]

```bash
git clone https://github.com/crossecute/forge-attest-example-aave-v4-steward
cd forge-attest-example-aave-v4-steward
git checkout --detach e6ce75bb1ec25abe5588d6e36ce41184ed38239f
git submodule update --init
git -C lib/aave-address-book submodule update --init lib/aave-v4
forge test
git clone https://github.com/crossecute/forge-attest.git ../forge-attest
git -C ../forge-attest checkout --detach v1.4.0
../forge-attest/attest.sh --config attest.toml --require-live
../forge-attest/attest.sh --config attest.child-a.toml --require-live
MAINNET_RPC_URL=YOUR_ETHEREUM_ARCHIVE_RPC_URL ../forge-attest/attest.sh --config attest.capture.toml --require-live
```

The three claims sit next to this study. All three passed a local v1.4.0 rerun with `--require-live --json`. The verdicts, Safe service records, execution receipt, fork block and CI run metadata are in [`evidence/`](evidence/README.md).

The service comparison checks the historical record for executed nonce 447. The capture config needs an archive RPC for fork block 26,337,525, and every `--require-live` check depends on the Safe service being reachable. The demo's CI uses the public `eth.drpc.org` endpoint.

This is not Aave's payload. The demo author could not find the code that produced nonce 447, and the steward repo's `diffs` folder holds only example reports. The values were decoded from the executed calldata. A match proves the transaction is exactly the output of this script at a pinned commit. It does not prove that Aave's own tooling produced it, that anyone reviewed the values, or that the steward accepts them. The research's match shows the Safe executed these calls, not that the team ran the steward script to propose them. [3, 7]

This study covers one transaction. It does not cover the other Aave Safes, the Aave Governance V3 proposal flow, or chains where the steward script only prints calldata.

## Sources

[[1] Safe service record for nonce 447](https://api.safe.global/tx-service/eth/api/v1/multisig-transactions/0x0ea9a8fed74bc267e88a5de190ca88a3c63b5963784f50176bfd0f4fb523f8e4/)

[[2] Executed Ethereum transaction](https://etherscan.io/tx/0x80169ec553baa63195fefd7ac61e6329879916284944f67fec7f82fd737dd473)

[[3] Demo repo, scripts, claims and test at commit `e6ce75b`](https://github.com/crossecute/forge-attest-example-aave-v4-steward/tree/e6ce75bb1ec25abe5588d6e36ce41184ed38239f)

[[4] Successful CI run, four jobs](https://github.com/crossecute/forge-attest-example-aave-v4-steward/actions/runs/42303115084)

[[5] Aave v4 risk steward proposal script](https://github.com/aave-dao/aave-v4-risk-stewards/blob/082760efc4e9d8b414d668fb80b3ec544b91b605/scripts/RiskStewardsBase.s.sol)

[[6] Forge Attest v1.4.0 documentation](https://github.com/crossecute/forge-attest/blob/v1.4.0/README.md)

[[7] Research identifying the Aave v4 stewards Safe and nonce 447](../../research/foundry-msig-proposal-research.md)
