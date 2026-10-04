# SafeEyes NG: Contracts

> Soroban smart contracts on Stellar for evidence anchoring, reporter stakes, reputation and escrowed stipends.

Part of the [SafeEyes NG](https://github.com/safeeyes-ng/docs) project: an open-source, verified crowd-sighting and alert network for kidnapping cases in Nigeria. Read the [docs repo](https://github.com/safeeyes-ng/docs) first for the mission, threat model and design principles.

> **Status:** Phase 0 / design. Nothing here is audited. **Do not deploy to mainnet or hold real funds until an independent security audit is complete.**

---

## Why Stellar, and why only these contracts

The alerts and media do **not** live on a blockchain. Stellar is used only where it adds something a normal database cannot:

| Contract | What it gives us |
|---|---|
| `evidence-anchor` | A tamper-evident, timestamped record that a file with a given hash existed and was not altered. Useful for police and courts |
| `reporter-stake` | A small refundable deposit that deters spam and malicious reports |
| `reputation` | A verifiable track record for reporters and verifiers, without exposing identities |
| `escrow` | Held stipends for verifiers and data contributors, released on verified work |

If a feature does not need a contract, it does not go on-chain.

## Contract overview

### `evidence-anchor`
Stores the SHA-256 hash of an evidence file with a ledger timestamp, tied to a case ID and an anchoring account.

- `anchor(case_id, hash)` records the hash and emits an event
- `get(hash)` returns the anchor record (case ID, ledger time, submitter)
- Duplicate hashes are rejected or flagged, to be decided in an ADR
- **Only hashes and identifiers go on-chain. Never files, names, phone numbers or locations**

### `reporter-stake`
- `deposit(reporter, amount, case_id)` locks a small stake
- `release(reporter, case_id)` returns the stake when the report is verified, called by an authorised verifier role
- `forfeit(reporter, case_id)` sends the stake to a defined sink when a report is confirmed malicious
- Stake sizes must stay small so low-income reporters are not excluded. Parameters are admin-configurable within bounds

### `reputation`
- Tracks counts of verified and rejected contributions per pseudonymous ID
- Exposes a score used by the backend to prioritise reliable sources
- Uses pseudonymous identifiers only, with no link to real-world identity on-chain

### `escrow`
- `fund(case_id or pool, amount)` holds stipend funds
- `release(recipient, amount)` pays verifiers and contributors for verified work
- **No bounties for rescues or confrontations.** Payments reward verification work and useful information only, never risk-taking

> The exact interfaces above are a starting design. Finalise each in an ADR and a spec file under `docs/` before implementing.

## Tech stack

- **Language:** Rust
- **Platform:** Soroban (Stellar smart contracts)
- **Tooling:** Stellar CLI (`stellar`), Cargo, `soroban-sdk`
- **Testing:** Soroban test utilities (in-process), plus testnet integration tests
- **Linting:** `cargo fmt`, `cargo clippy`
- **CI:** build, test, clippy, and WASM size check on every PR

Check the current Soroban documentation for exact SDK versions and CLI commands, as they change between releases.

## Repository layout

```
contracts/
├── Cargo.toml                  # workspace
├── contracts/
│   ├── evidence-anchor/
│   │   ├── Cargo.toml
│   │   └── src/{lib.rs, test.rs}
│   ├── reporter-stake/
│   ├── reputation/
│   └── escrow/
├── tests/
│   └── integration/            # testnet scenario tests
├── specs/                      # per-contract specs and invariants
├── scripts/                    # build, deploy (testnet), invoke helpers
├── audit/                      # audit reports and responses (when available)
├── .env.example
├── CLAUDE.md
└── README.md
```

## Getting started

### Prerequisites
- Rust (stable) with the `wasm32` target for Soroban (see Soroban docs for the exact target name for your CLI version)
- Stellar CLI
- A funded **testnet** account (use the testnet friendbot)

### Setup
```bash
git clone https://github.com/safeeyes-ng/contracts.git
cd contracts
cargo build
cargo test
```

### Build and deploy to testnet
```bash
stellar contract build
# Create/fund a testnet identity, then deploy:
stellar contract deploy --wasm <path-to-wasm> --source <identity> --network testnet
```

Exact flags vary by CLI version. Confirm against the current Stellar docs and record working commands in `scripts/`.

### Keys and networks
- **Testnet only** during development. Never use mainnet keys in a dev environment or Codespace
- Never commit secret keys, seed phrases or `.env` files
- Production keys must live in a hardware wallet or managed secrets system with multisig, decided in an ADR before the pilot

## Security requirements

This repo has the strictest review rules in the project.

1. **Specs and invariants before code.** Each contract has a spec in `specs/` listing its invariants (for example: *a stake cannot be released twice*, *only the verifier role can release*).
2. **Every invariant has a test** that fails if broken.
3. **Authorisation checks on every state-changing function** using Soroban's auth mechanisms. No implicit trust.
4. **No personal data on-chain.** Hashes, pseudonymous IDs, amounts and case IDs only.
5. **Admin powers are minimal, bounded and transparent,** with a plan for multisig and, where appropriate, timelocks.
6. **Upgrade strategy documented** in an ADR before any mainnet deployment.
7. **Two reviewers** required on every PR. At least one with Rust/Soroban experience.
8. **Independent audit** before mainnet and before real funds. Publish the report in `audit/`.
9. **Bug bounty or coordinated disclosure** plan before launch. Report vulnerabilities privately via the docs repo `SECURITY.md`.

### Threats to consider
- Unauthorised release or forfeit of stakes
- Replay or duplicate anchoring
- Griefing: forcing victims' reporters to lose stakes
- Reputation manipulation or Sybil attacks
- Admin key compromise
- Denial of service through state or storage bloat
- Linkability: pseudonymous IDs being correlated with real identities off-chain

## Testing strategy

- **Unit tests** in each contract's `test.rs` using Soroban's test environment
- **Property and fuzz tests** for stake accounting and escrow balances where feasible
- **Scenario tests** on testnet: full case flow (deposit, anchor, verify, release)
- **Negative tests:** wrong caller, double release, zero amounts, overflow, expired cases
- CI fails on any warning from `clippy` or a failing test

## Integration with the backend

The `backend` repo's `anchoring` module calls these contracts through the Stellar SDK. Keep the contract interfaces stable and versioned. Publish ABI/spec changes in `specs/` and note them in the changelog so the backend team can update.

## Contributing

1. Read the [docs repo CONTRIBUTING.md](https://github.com/safeeyes-ng/docs/blob/main/CONTRIBUTING.md) and this README's security section.
2. Pick an issue labelled `good first issue`, `stellar` or `help wanted`, and comment to claim it.
3. For any new contract or interface change, open a **spec PR first** and wait for approval before implementing.
4. Branch naming: `feat/…`, `fix/…`, `docs/…`. Conventional commits.
5. Include tests. PRs need two approvals and passing CI.

**Good first issues to open:**
- Set up the Cargo workspace, CI and `clippy`/`fmt` checks
- Write the `evidence-anchor` spec and invariants
- Implement `evidence-anchor` with unit tests
- Write the `reporter-stake` spec, including edge cases for forfeit
- Script for testnet deploy and invoke
- Add a WASM size check to CI
- Threat-model review of the stake contract for griefing scenarios

## Working with Claude Code in a Codespace

1. Open the repo in a Codespace. Add a `.devcontainer/devcontainer.json` with Rust and the Stellar CLI.
2. Install Claude Code (`npm install -g @anthropic-ai/claude-code`, or see Anthropic's docs for the current method) and run `claude` in the repo root.
3. Create a `CLAUDE.md` with:
   - The mission and what the contracts are for
   - The **Security requirements** section above, verbatim
   - "Testnet only. Never write or request real secret keys"
   - "Write the spec and invariants first, then tests, then code"
   - "No personal data on-chain"
   - Pointers to the current Soroban docs, since APIs change
4. Example first prompts:
   - *"Create the Cargo workspace and a skeleton `evidence-anchor` contract with an `anchor` and `get` function and unit tests."*
   - *"Write `specs/reporter-stake.md` listing invariants and failure cases before any code."*
   - *"Review `reporter-stake` for authorisation gaps and griefing scenarios and list findings."*
5. **AI-written smart contract code is untrusted until a qualified human reviews it and an independent audit covers it.** Use Claude Code to draft, test and review, not as the final authority.

## Security

Never open a public issue for a vulnerability. See `SECURITY.md` in the docs repo for private reporting.

## License

Apache-2.0 (confirm and add a `LICENSE` file before the first release).
