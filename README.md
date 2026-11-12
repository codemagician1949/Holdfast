# Holdfast

[![ci](https://github.com/codemagician1949/Holdfast/actions/workflows/ci.yml/badge.svg)](https://github.com/codemagician1949/Holdfast/actions/workflows/ci.yml)

**Holdfast turns a NEAR name into a small business you can sell.** An AI service lives inside the name and earns from paid jobs. When you sell the name, the buyer gets the business, the money it earns from then on and its public track record, and every key or AI agent the seller left behind stops working. The chain enforces that, not a promise from the seller.

- **Live app:** https://holdfast-near.vercel.app · **Network:** NEAR **testnet** only · **License:** MIT
- **Tracks:** `nfts` (lead) · `security` · `agents` ([how each track's prompt is answered](docs/DETAILS.md#track-fit))
- **Formerly named Deed.** Renamed on 12 Nov 2026 because another NEAR Hackathon project, by a different team, is also called Deed (an NFT-owned keyless account with drain protection); the two were built independently and share no code. On-chain account names still say `deed` (`deed.deedhq.testnet`, `*.deedbot.deedhq.testnet`), because renaming them would mean redeploying and losing the recorded evidence. The repo moved from github.com/codemagician1949/Deed (old links redirect), and deed-near.vercel.app still works.

## The problem, in plain words

Selling an online business or an AI agent today means handing over logins, API keys and wallet access, and trusting the seller not to keep a copy. On NEAR, a name can be an NFT and a full account at the same time, and the account's own code can take the seller's access away when the NFT changes hands. Holdfast puts a working, paid AI service inside such a name, so selling the NFT sells the business.

The service in the demo is **Preflight**: before you or your AI agent sign a NEAR transaction, it explains it in plain words and flags anything risky. Each answer costs 0.05 test NEAR and is written and signed inside a secure enclave on NEAR AI.

## What Holdfast built, and what it builds on

**Builds on (not ours):** House of Stake's **Top Level Accounts** make a rented name both an NEP-171 NFT and a keyless account, and wipe every agent permission when the name changes owner ([source](https://github.com/houseofstake/tla-contracts); audited by Valhalla Security according to [their README](https://github.com/houseofstake/tla-contracts#status); live on mainnet as [`registry.hos`](https://nearblocks.io/address/registry.hos)). It isn't deployed on testnet, so we run the **byte-identical mainnet bytecode** under team accounts, and the checker compares the code hashes with mainnet live.

**Holdfast adds (ours, not audited):**
- **Activation registry:** the owner switches the service on under the name, and only the current owner can.
- **Sale escrow with royalties:** the buyer pays the exact price; the name moves and the seller is paid in the same transaction, minus a 2.5% royalty; if anything fails, the buyer gets everything back.
- **Job escrow:** a customer's payment is held by the contract until the answer is delivered, and refunded if it isn't.
- **The Preflight runtime on NEAR AI's TEE:** it re-checks the owner on every job, and its answers carry an enclave signature anyone can verify.
- **Counterparty checks for agents that pay:** a free allow / warn / block check, a shared blocklist contract, and an agent vault that refuses reported receivers on chain ([details](docs/DETAILS.md#for-agents-that-pay-track-security)).
- **The web app**, and a **verifier** that attacks every sale from the seller's side and checks each claim on chain.

## See it in 2 minutes (browser, no wallet)

1. **https://holdfast-near.vercel.app/deed/preflight.deedbot.deedhq.testnet**: every line under **History** is a testnet transaction: two sales, the wallet wiping the seller's permissions, earnings paid out, and each refused attempt by the old owner with the chain's own error. **Jobs answered** lists paid answers with their enclave-signature checks.
2. **https://holdfast-near.vercel.app/proof**: the live sale's attack run, one row per attempt, each linked to its transaction.
3. **https://holdfast-near.vercel.app/try**: click an example transaction and watch the free risk flags and the check of who it pays.

## Try it yourself (5 minutes, free test coins)

1. Install a NEAR wallet such as Meteor or MyNearWallet, switch it to **testnet**, and get free test NEAR from [near-faucet.io](https://near-faucet.io).
2. On [Try Preflight](https://holdfast-near.vercel.app/try), click **Approve unlimited token spending**. You see a high-risk flag before paying anything.
3. Connect your wallet, pick a Holdfast, click **Pay 0.05 NEAR and explain**, and approve in your wallet.
4. **You see** the card go from *Answering…* to *Answered*, with a verdict (here *DO NOT SIGN*) and two ticks under the enclave signature. On that Holdfast's page, your payment appears under **Jobs answered** and its earnings go up by 0.05. If anything fails, the contract refunds you, and after 15 minutes anyone can trigger the refund.

## How it works

1. **Rent.** A name rented from the House of Stake registry becomes a keyless account run by the global `hos-wallet` code; only extensions (owner, lease authority, agents) can act.
2. **Activate.** The owner calls `deed.activate(name, runtime_id, config_hash, price, version, runtime_url)`; the contract checks the caller against the registry owner in a callback and keeps a per-name version high-water mark, so an old activation can never be replayed. `config_hash` is the sha256 of the public service config.
3. **Earn.** A customer calls `deed.pay_job(name, request_hash)` with the price: the **contract holds the money**, bound to that exact request and to the runtime the owner activated. The runtime answers, then calls `settle_job`; the contract re-reads the registry owner and pays the name's own wallet only if the job's owner still owns it, otherwise it refunds the customer. The runtime can refund at once, and **anyone** can refund a job left unsettled for 15 minutes, so refunds never depend on the operator. The runtime's only key is a function-call key limited to `settle_job`/`refund_job`.
4. **List.** `nft_transfer_call` moves the name into the escrow. On that rotation the wallet **clears spend grants, removes every extension and sweeps the NEAR balance to the seller**.
5. **Buy.** `deed.buy` (exact price) re-points payout to the seller; only if that succeeds does it transfer the name. In the same callback the seller is paid the price minus royalty, or, if the transfer failed, the buyer is refunded in full. No waiting window: House of Stake's code has no way to undo a completed transfer (read from its source), and a name that ends up back in the escrow is returned to the buyer with `recover_name`, never refunded.
6. **Switch on.** The buyer activates with a higher version; the next paid answer lands in the same name under the new owner.


Call-by-call sequence, contract API, runtime API and web routes: [docs/REFERENCE.md](docs/REFERENCE.md).

## What a sale guarantees

After a sale, enforced on chain: the seller's agent permissions and extensions are gone (G1); the seller can't sign in or act as the name (G2); what the name earned under the seller went to the seller, once (G3); the seller's activation and jobs paid before the sale can't pay the sold name (G4); the buyer paid exactly the price and the seller got it in the same transaction, or the buyer was refunded (G5). Each is attacked by the verifier: [the claim, who enforces it and where it's proven](docs/DETAILS.md#the-exact-security-claim) · [threat model, 43 rows](THREAT_MODEL.md) · [every guarantee by whose code](docs/INVARIANTS.md).

**Not claimed:** that the seller forgets what they knew, that an old operator's server can't answer (it just can't be paid), or that an answer is correct (the enclave signature proves it is authentic).

## Verify it from a clean clone

```bash
git clone https://github.com/codemagician1949/Holdfast && cd Holdfast
pip install dcap-qvl==0.7.1               # optional: verifies the Intel TDX quote locally (without it the checker asks Phala's hosted verifier)
python3 verify/check-testnet.py --runtime https://preflight-production-8802.up.railway.app   # ~1-3 min, no wallet, no NEAR
contract/test.sh                          # ~2-5 min (fetches the NEAR sandbox itself): 36 tests (`cargo test` prints 3+17+4 for deed, 5 reports, 6+1 agent-vault): deed 24 (21 in a local NEAR sandbox, 4 of those on House of Stake mainnet bytecode), reports 5, agent-vault 7 (1 in the sandbox with a real function-call key)
python3 verify/recompute-title.py --compare  # ~1 min: recomputes the title page's numbers from chain data (jobs and sales cross-checked against contract state) and compares with the page's own code
verify/rebuild-deed.sh                    # ~5 min, needs Docker: rebuilds the deployed Holdfast contract from the commit in its NEP-330 metadata and compares the hash
cd runtime && bun install && bun test     # seconds: 51 tests, prints `51 pass` (escrowed job rules, settle/refund on chain, sale mid-job, crash recovery, TEE signature, risk flags, counterparty check)
```

**You will see** `ALL CHECKS PASS`: mainnet and testnet code hashes match, every transaction in the published sale report exists with the outcome it claims, the hosted runtime is the one the chain names, and NEAR AI's enclave attestation verifies independently. [What each check proves, and the reconciled numbers](docs/DETAILS.md#what-the-checker-verifies-and-the-numbers).

## What is real today

Live on testnet: the contracts, two AI runtimes, the web app, and a real sale where every attack by the old owner was refused on chain ([evidence](docs/EVIDENCE.md)). All activity so far is between the team's own testnet accounts, so it shows the mechanism works, not that anyone else wants it. Not yet done: a buyer from outside the team running their own runtime ([runbook](docs/BUYER-RUNBOOK.md)), and a risk-flag accuracy test labelled by an outsider. Full table: [what is real and what isn't](docs/DETAILS.md#what-is-real-today-and-what-isnt).

## Glossary

| Word | Meaning |
|---|---|
| **TLA** (Top Level Account) | A name rented from House of Stake's registry, e.g. `preflight.deedbot.deedhq.testnet`. It is an NFT and a NEAR account at once. |
| **Keyless account** | An account no person holds a key for; only its code and its listed extensions can act for it. |
| **Extension** | A permission on the name's wallet: the owner, the lease authority, or an AI agent with a spending allowance. |
| **Rotation** | The wallet's change of owner when the name is sold or transferred. It removes every extension and spending allowance and pays the balance out to the previous owner. |
| **Activation** | The owner's on-chain record that switches the service on: which runtime, which config, which price. |
| **Runtime** | The server that answers paid jobs for a name. It can only be paid if the current owner's activation names it. |
| **Escrow** | The Holdfast contract holding money (a job payment or a sale price) until its conditions are met. |
| **TEE / enclave** | Hardware that runs the AI model and signs its answers, so anyone can check an answer wasn't changed. |
| **NEP-641, NEP-413** | NEAR standards for signing in as a name and for signed messages; the owners-only console uses them. |

## Prior art

ERC-6551 gives each NFT a separate account whose control follows the token; on NEAR the token and the account are the same name, and a sale also wipes delegated agent permissions and settles the balance to the seller. ERC-7857/0G iNFTs re-encrypt agent secrets on transfer through a TEE or oracle; Holdfast keeps nothing secret, and control passes because the wallet wipes extensions on rotation. Virtuals tokenises speculation on an agent; Holdfast sells the operating business (name, earnings, record). Flippa-style marketplaces rely on a trusted handover; here it is enforced on chain and attacked by a verifier.

## Built with

- **Libraries:** Rust `near-sdk` 5.30, `near-workspaces` 0.24; Bun with `@noble/curves` and `@noble/hashes`; React 19, Vite, Tailwind 4, shadcn/ui (Base UI), motion, react-router, `@hot-labs/near-connect`. Services: NEAR RPC, FastNEAR APIs, NEAR AI Cloud. NEAR features used, with file paths: [docs/DETAILS.md](docs/DETAILS.md#near-features-used).
- **Third-party code deployed unmodified:** House of Stake [`tla-contracts`](https://github.com/houseofstake/tla-contracts) mainnet bytecode, vendored with its hashes in [deploy/mainnet-wasm](deploy/mainnet-wasm/).
- **AI-assisted development (Q-005):** the code, tests, scripts and docs were written with an AI coding assistant (Claude Code), directed and reviewed by codemagician. Design critiques we asked AI models for are in [docs/process/](docs/process/): working notes, not external validation.
- **Baseline (Q-003):** everything here was built during the hackathon, from 2026-11-02. The tag `pre-hackathon` marks the first commit ([9aab65e](https://github.com/codemagician1949/Holdfast/commit/9aab65e)) and `hackathon-submission` the final one: [everything built during the hackathon](https://github.com/codemagician1949/Holdfast/compare/pre-hackathon...hackathon-submission).

## Team and other submissions

- **Team:** codemagician ([codemagician1949](https://github.com/codemagician1949) on GitHub), a team of one.
- **Other projects in this hackathon (Q-015):** none; Holdfast is codemagician's only submission. It shares no code, accounts or servers with any other project.

## Documents

[docs/DETAILS.md](docs/DETAILS.md) (full claim, track fit, agents, numbers) · [THREAT_MODEL.md](THREAT_MODEL.md) · [docs/INVARIANTS.md](docs/INVARIANTS.md) · [docs/EVIDENCE.md](docs/EVIDENCE.md) (live sale, live jobs, bugs found) · [docs/REFERENCE.md](docs/REFERENCE.md) (run it, call-by-call, APIs) · [docs/CLIENTS.md](docs/CLIENTS.md) · [docs/BUYER-RUNBOOK.md](docs/BUYER-RUNBOOK.md) · [docs/SPIKE.md](docs/SPIKE.md) · [docs/RESEARCH.md](docs/RESEARCH.md)
