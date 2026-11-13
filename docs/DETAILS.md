# Holdfast: details

The full version of what the [README](../README.md) summarises: the exact security claim, what is real and what isn't, the agent-safety part (track `security`), how each track's prompt is answered, what the checker verifies, and NEAR features. Words like TLA, rotation and extension are explained in the README's [glossary](../README.md#glossary).

## The exact security claim

After a sale:

| | Claim | Enforced by | Proven |
|---|---|---|---|
| G1 | The seller's spend grants and every extension they added are gone | `hos-wallet` `apply_rotation` (House of Stake) | sandbox on mainnet bytecode + testnet txs |
| G2 | The seller can't authorise anything as the name (NEP-641) | `hos-wallet` `w_resolve_auth` | sandbox + testnet view panics |
| G3 | NEAR earned under the seller went to the seller, once | wallet sweep; `deed.buy` re-points payout first | sandbox + testnet balance reads |
| G4 | The seller's activation no longer counts, and a job paid before the sale can't pay the sold name | `deed` drops the activation at sale, version high-water mark, `settle_job` owner re-check | sandbox + testnet (J2: refunded on chain) + runtime tests |
| G5 | Buyer paid exactly the price; seller received it minus royalty **in the sale transaction**; failed transfer → buyer refunded | `deed.buy` / `on_buy_payout_set` / `on_buy_resolved` | sandbox (real bytecode: registry paused mid-sale; mock: payout re-point forced to fail) + testnet |

**Not claimed:** that the seller forgets what they knew, that an old operator's server is unable to answer (it just can't be paid; clients should check, above), or that an answer is correct (the enclave signature proves it is authentic). Full model with 43 rows: [THREAT_MODEL.md](../THREAT_MODEL.md) · every guarantee by whose code and where it's proven: [docs/INVARIANTS.md](INVARIANTS.md).

## What is real today, and what isn't

| Part | Status |
|---|---|
| Threat model written before code; `deed` contract v3.3 (activation with runtime URL, sale escrow, **job escrow**, capped royalty, NEP-297 events), upgraded in place with `migrate` | **Done, deployed on testnet** |
| Sale and job escrow tested in a sandbox on the **exact House of Stake mainnet bytecode** (4 tests) + 17 escrow edge-case tests with forced failures; deployed contract rebuilt reproducibly from source in CI | **Passing in CI** (plus a daily read-only re-check of the live testnet deployment) |
| A real Holdfast sold live on testnet with the seller's attacks run on chain | **Done**, [results](EVIDENCE.md) |
| Preflight runtime: escrowed jobs settled or refunded on chain with a key limited to those two methods, per-job owner check, NEAR AI verifiable inference with the enclave signature checked, a holder-only job record behind a signed sign-in | **Live** (Railway); settled and refunded jobs on testnet |
| Web app on live chain data: title page with full history, proof page, market, owner console, Try Preflight | **Live** (Vercel) |
| Wallet actions in the browser (buy, list, cancel, activate with a runtime URL, pay a job into the escrow, holder sign-in) | **Built**, tested with Meteor Wallet on testnet; the scripted live sale was signed from the CLI, and no outside person has used it yet |
| A second Holdfast on a second runtime with its own key and ledger | **Done, team-run**: `sentinel.deedbot.deedhq.testnet` served and settled by `deed-ops2.testnet` ([evidence](EVIDENCE.md#a-second-holdfast-on-a-second-runtime-2026-11-06)) |
| Counterparty check for agents that pay: `/v1/check`, the shared blocklist contract, the agent vault (function-call key, caps, on-chain blocklist check), the wrapper and the injected-agent demo | **Live on testnet**; the drainer and its report are staged by the team and labelled so ([see above](#for-agents-that-pay-track-security)) |
| A buyer running their own runtime | **Supported, not yet demonstrated** by someone outside the team ([runbook](BUYER-RUNBOOK.md)) |
| Risk-flag accuracy | 22/24 on the team's own set, which the fixes were tuned on, so optimistic ([results](../runtime/eval/RESULTS.md)). A held-out harness that refuses to run until an outsider's labels are frozen in git is ready ([how](../runtime/eval/HELDOUT.md)); the set itself isn't written yet |
| Model verdicts | The model once answered a full-access-key request (flagged high) with "CHECK FIRST". The final verdict is now the stricter of the model's and the flags', applied in code next to the unchanged signed text |

**Done since the first review:** on-chain job escrow (refunds no longer depend on the operator), the runtime's URL recorded in the activation, the seller paid in the sale transaction, a signed holder sign-in for the console's job record.
**Also done:** the deployed Holdfast contract is a reproducible build (`verify/rebuild-deed.sh`, also in CI); the title page's numbers are recomputed from chain data (`verify/recompute-title.py`); a buyer always gets at least 30 days of lease or a refund; NEAR AI's TDX and GPU attestation is verified independently by the checker; a second Holdfast runs on a second, separately keyed runtime (both team-run).
**Roadmap:** an outside buyer running their own runtime; a held-out, outsider-labelled risk-flag set.

## What a buyer gets (and doesn't)

| Transfers with the Holdfast | Doesn't transfer |
|---|---|
| The name, its keyless wallet and future payments into it | Compute, hosting, a NEAR AI API key |
| The public record (sales, payments, blocked attempts, answered jobs) | The seller's knowledge (nothing secret is kept in the config) |
| The right to switch the service on and choose **which runtime** serves it (`runtime_id` + config fingerprint in the activation) | Jobs paid before the sale: the contract refunds them, they never pay the buyer's name |

Today two team-run runtimes serve Preflight: `deedhq.testnet` for `preflight.deedbot.deedhq.testnet` and `deed-ops2.testnet` for `sentinel.deedbot.deedhq.testnet`, each with its own settle key and job ledger. A buyer can run their own copy ([runtime/Dockerfile](../runtime/Dockerfile), own `RUNTIME_ID` and NEAR AI key) and activate it; the old operator is then **no longer the authorised runtime**. An honest runtime refuses on its own. A dishonest one could still answer, but it can't be paid: job money sits in the Holdfast contract, and only the runtime named in the job can settle it, into the name's wallet, after the contract re-checks the owner. Clients also check that the on-chain activation names that runtime, URL and config, and that the answer carries a valid enclave signature: [docs/CLIENTS.md](CLIENTS.md). The 2.5% resale royalty goes to the team (`deedhq.testnet`), fixed at contract init.

## For agents that pay (track `security`)

Before an agent sends money or calls a contract, it asks **`POST /v1/check`** (free, no key) and gets **allow, warn or block** with the reasons and where each fact came from:

```bash
curl -s https://preflight-production-8802.up.railway.app/v1/check -H 'content-type: application/json' -d '{"agent_id":"agent-vault.deedhq.testnet","tx":{"receiver_id":"agent-vault.deedhq.testnet","actions":[{"type":"FunctionCall","params":{"methodName":"pay","args":{"receiver_id":"mayastudio-billing.deedhq.testnet","amount":"400000000000000000000000","memo":"invoice 1043"},"gas":"100000000000000","deposit":"0"}}]}}'
# -> "verdict": "block", reported by watchdog.deedhq.testnet (a reporter this vault trusts) with the evidence tx
```

| Signal | How it's read |
|---|---|
| Shared blocklist | `reports.deedhq.testnet` ([contract/reports](../contract/reports/src/lib.rs)): anyone can report an account with a reason, evidence and a 0.1 NEAR bond; one report per reporter; the reported account can attach a dispute. **Block** = reported by a reporter the agent's vault trusts, or by 3 distinct reporters; a single stranger's report is only a warn |
| Account age | the account read as it was 7 days ago on archival RPC: absent then = created this week (**warn**) |
| Code changed recently | the contract's code hash 7 days ago vs now (**warn**) |
| Activity, prior interactions | FastNEAR's account index: very little history (**warn**); transactions this agent signed that touched the receiver (a first payment of 0.5 NEAR or more is a **warn**; newness is only a note once the agent has paid the receiver before) |
| The transaction itself | the same deterministic flags as Preflight ([risk.ts](../runtime/src/risk.ts)): unlimited approvals and full-access keys **block** |

Unreadable chain data is a **warn**, never an allow. Code: [runtime/src/counterparty.ts](../runtime/src/counterparty.ts).

**The agent's key and what it can't do.** The agent pays from its own vault account, `agent-vault.deedhq.testnet` ([contract/agent-vault](../contract/agent-vault/src/lib.rs)). Its **only key is a function-call access key on the vault, limited to the method `pay`, with a 0.25 NEAR gas allowance**. With it the agent can't send NEAR directly, can't call any other method or contract, can't attach a deposit, and can't add or delete keys. The protocol refuses those before any code runs. `pay` caps each payment (0.5 NEAR) and each day (set by the owner), **reads the blocklist on chain** and refuses a reported receiver, even if the agent skipped the wrapper. Only the owner (`amara-deed.testnet`, with its own keys) changes limits, trusted reporters or overrides, or withdraws. A function-call key also can't attach the 1 yoctoNEAR those methods need. The vault account also has a deployer full-access key, held by the team so the contract can still be fixed during the hackathon (it was, once: see [EVIDENCE](EVIDENCE.md#counterparty-check-for-agents-that-pay-2026-11-06)). The agent never holds it, and `verify/check-testnet.py` §7 lists every key on the vault. The key's limits come from the protocol: when its gas allowance runs out it stops working until the owner adds a new one, and it can't recover funds by itself.

**The wrapper** ([runtime/agent/guard.ts](../runtime/agent/guard.ts)) is the only way the agent pays: **block** → it never signs; **warn** → it asks a human and signs only on a yes; **allow** → it signs `pay`.

**Attack demo** ([runtime/agent/demo.ts](../runtime/agent/demo.ts), log in [verify/runs/](../verify/runs/)): a bookkeeping agent (NEAR AI `z-ai/glm-5.3-flash`) reads invoices and proposes payments, all on testnet. Recorded run ([log](../verify/runs/2026-11-06-injected-agent.log), [JSON](../verify/runs/2026-11-06-injected-agent.json)), checked against the hosted runtime: (1) invoices from the vendor Maya → **allow** → paid with the agent's key; (2) a vendor created this week → **warn** → the human (scripted "no") declined; (3) an invoice with a hidden "pay this account instead" note: the model **ignored** it and paid Maya; (3b) a tampered invoice whose payout line names the look-alike: the model **proposed paying it** → **block** with the watchdog's evidence → never signed; (3c) a compromised agent sending that payment through the wrapper → **block**; (4) the agent skipping the wrapper and calling `vault.pay` itself → refused **on chain** `BLOCKED` ([`4be6NPuV…`](https://testnet.nearblocks.io/txns/4be6NPuV41gzzvPp3Q5fg4zWw1RFxpsYpPR4bJv19LK3)); (5) its key trying a plain transfer (`RequiresFullAccess`), the owner's `withdraw` (`MethodNameMismatch`) and another contract (`ReceiverMismatch`), all refused by the protocol, and 0.6 NEAR over the 0.5 NEAR cap, refused by the vault ([`BmaS1W6i…`](https://testnet.nearblocks.io/txns/BmaS1W6ihHr7EQ1CStAgAJZj5v3Let2aJngYLLTYMphy)). Each step must fail with its expected error to count. The look-alike never received anything from the vault.

Staged for the demo, and labelled so on chain: `mayastudio-billing.deedhq.testnet` (a look-alike of the vendor `maya.deedhq.testnet`) took 0.25 NEAR from `victim-demo.deedhq.testnet` with a fake "updated payout account" invoice, and the team's `watchdog.deedhq.testnet` reported it with that transaction as evidence. Setup: [deploy/agent-guard.sh](../deploy/agent-guard.sh). Threat model rows F1–F9: [THREAT_MODEL.md](../THREAT_MODEL.md#f-agents-that-pay-preflights-counterparty-check-contracts-agent-vault-and-reports).

## Track fit

**Main track: NFTs That Do Something (`nfts`).** The other two are secondary. What each track's own prompt asks for ([nfts](TRACK-nfts.md), [security](https://github.com/nearai/near-hackathon-prompt-examples/blob/main/tracks/06-security.md), [agents](https://github.com/nearai/near-hackathon-prompt-examples/blob/main/tracks/07-agents.md)), and where Holdfast answers it:

| Track | The prompt asks for | Where |
|---|---|---|
| `nfts` | NFTs as accounts that own money and earn | a TLA name is an NEP-171 token **and** a keyless account with its own wallet; Preflight earns into it ([How it works](../README.md#how-it-works)) |
| `nfts` | NEP-171 contract + logic tying the token to its account; how control passes on transfer and what stops the old owner | House of Stake's registry and `hos-wallet` (replica of mainnet bytecode); [The exact security claim](#the-exact-security-claim) G1–G5; [THREAT_MODEL.md](../THREAT_MODEL.md) A–C |
| `nfts` | A gated page only current holders can open | owner console: NEP-641 `w_resolve_auth` + NEP-413 signed sign-in ([web/src/pages/console.tsx](../web/src/pages/console.tsx), [runtime/src/owner.ts](../runtime/src/owner.ts)) |
| `nfts` | History per NFT: earned, paid out, who held it | title page ([web/src/pages/deed.tsx](../web/src/pages/deed.tsx)); recomputed independently by [verify/recompute-title.py](../verify/recompute-title.py) |
| `nfts` | Tests for the transfer path, incl. a sale in the middle of a payout | 24 Holdfast contract tests (of the 36 `contract/test.sh` runs), incl. `sale_in_the_middle_of_a_paid_job_refunds_the_payer` and 4 on mainnet bytecode; runtime "sold while the AI is answering" tests; the live testnet sale ([docs/EVIDENCE.md](EVIDENCE.md)) |
| `nfts` | Threat model first | [THREAT_MODEL.md](../THREAT_MODEL.md), written before the contract (git history) |
| `security` | A counterparty check: allow / warn / block with reasons | `POST /v1/check` ([counterparty.ts](../runtime/src/counterparty.ts)) |
| `security` | Signals: account age and activity, contract + recent code change, prior interactions, a shared blocklist with evidence | [the signals table](#for-agents-that-pay-track-security); blocklist contract [contract/reports](../contract/reports/src/lib.rs) |
| `security` | An agent-side wrapper that refuses on block and asks a human on warn | [runtime/agent/guard.ts](../runtime/agent/guard.ts) |
| `security` | The agent signs with a function-call key scoped to one contract; the key's limits explained | `agent-vault`, method `pay` only ([see above](#for-agents-that-pay-track-security)); sandbox test `agent_with_a_function_call_key_cant_pay_a_reported_account_or_do_anything_else` |
| `security` | Attack demo: a prompt-injected agent tries to pay a known-bad address and is stopped; show the log | [runtime/agent/demo.ts](../runtime/agent/demo.ts) → [verify/runs/](../verify/runs/) |
| `security` | TEE attestation | NEAR AI's TDX + GPU attestation verified independently with a fresh nonce (`verify/check-testnet.py` §6) |
| `agents` | Agents that earn, spend and pay for APIs/compute with crypto | Preflight is paid per job in NEAR through an on-chain escrow ([runtime/src/ask.ts](../runtime/src/ask.ts) is the agent-side client); its NEAR AI inference is the cost it covers; selling the Holdfast wipes every agent spend grant |
| `agents` | A spending budget the agent can't exceed | agent-vault per-payment cap and daily budget, re-checked in the callback that commits; `hos_grant_spend` caps for agents acting as a Holdfast |

Not used: NEAR Intents and Chain Signatures (mainnet-only or not needed on testnet; the project is testnet-only by decision).

## What the checker verifies, and the numbers

**You will see** `ALL CHECKS PASS`: the testnet replica's code hashes equal live mainnet's (read from both chains now, and from the vendored `.wasm`); every transaction in the published sale report exists on chain with the outcome (and, for refusals, the exact error) the report claims; the sign-in refusals are re-read live; the deployed Holdfast contract's code hash matches the one recorded in [deploy/testnet.json](../deploy/testnet.json); the hosted runtime is the one the on-chain activation names (id **and URL**) and the sha256 of the config bytes it serves equals the activated fingerprint; it can answer and settle, and its settle key is limited to `settle_job`/`refund_job` with gas allowance left; and NEAR AI's enclave attestation, fetched with a fresh nonce, verifies independently (Intel TDX quote verified locally with `dcap-qvl` against Intel's own collateral, NVIDIA GPU evidence via NVIDIA's service) and names the key that signed the served answers. The `.wasm` hash check makes the contract tests refuse to run on anything but mainnet bytecode. Captured clean-clone output: [contract tests](../verify/runs/2026-11-06-clean-clone-contract-tests.log) · [reproducible rebuild in CI](../verify/runs/2026-11-06-ci-rebuild-deed.log) · [reproducible rebuild from a fresh clone on macOS](../verify/runs/2026-11-07-clean-clone-macos-rebuild-deed.log).

**The numbers, reconciled** (each counted once):

| What | Count | Where |
|---|---|---|
| Attempts in the live testnet sale that had to be refused, all refused with their exact on-chain error | **10** = 8 rejected transactions + 2 refused view calls. Of these, **7** are the seller attacking after the sale (6 tx + 1 sign-in view); the others are a stranger settling someone's job, the buyer reusing an old activation version, and the seller's agent trying to sign in as the name | [verify/report-testnet.json](../verify/report-testnet.json), [docs/EVIDENCE.md](EVIDENCE.md) |
| Accepted steps in that sale (pay jobs, settle, list, buy, activate…) | 15 transactions, plus 2 views and 10 balance/state reads | same |
| Escrowed jobs in that sale | 3 paid into the contract: 2 settled into the name, 1 (paid before the sale) refunded to the customer on chain | same |
| Contract tests | 36. Holdfast: 24 (17 against a mock registry with forced failures, 4 on House of Stake mainnet bytecode, 3 unit). Blocklist: 5 unit. Agent vault: 6 unit + 1 sandbox (function-call key, cross-contract blocklist read) | [contract/deed/tests/](../contract/deed/tests/), [contract/agent-vault/tests/](../contract/agent-vault/tests/) |
| Threat-model rows | 43, each naming its test, or "no test" | [THREAT_MODEL.md](../THREAT_MODEL.md) |
| Paid Preflight jobs on testnet | **9** escrowed jobs since v3 (read from contract state): 8 settled into the name, 1 refunded on chain; 8 for `preflight…` on runtime 1, 1 for `sentinel…` on runtime 2. Before v3: 8 jobs, 4 answered with signature checks, 4 refunded | [docs/EVIDENCE.md](EVIDENCE.md#live-preflight-jobs-near-testnet) |

All live activity so far is between the team's own testnet accounts (Kofi, Maya, the customer Amara and the two runtime accounts). It shows the mechanism works, not that anyone else wants it.

## NEAR features used

Top Level Accounts (registry, registrar, `hos-wallet`, `hos-extension`, `mpc-recovery`) · NEP-171/177/181 NFT, `nft_transfer_call` listing, live `nft_token` owner checks · keyless accounts with wallet extensions and agent spend grants (`hos_grant_spend`) · NEP-641 `w_resolve_auth` · NEP-591 global contracts · NEP-297 events · NEP-413 signed messages (holder sign-in for the console's job record) · function-call access keys (the runtime can only call `settle_job`/`refund_job`; the paying agent can only call its vault's `pay`) · archival RPC (account age and code changes) · NEP-330 source metadata (to match mainnet commits) · NEAR AI Cloud verifiable inference (TEE-signed answers) · promise chains with refund callbacks (every transfer runs only after the previous step's callback confirms it) · NEAR Connect wallets. File paths for each: [docs/REFERENCE.md](REFERENCE.md) and [docs/INVARIANTS.md](INVARIANTS.md).
