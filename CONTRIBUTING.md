# Contributing to Ndi-Fi

Thanks for helping build Ndi-Fi. This guide covers contribution rules, workflows,
and review expectations across the repository, including our participation in the
[Stellar Wave Program on Drips](https://www.drips.network/wave/stellar).

> 🐣 **New contributor?** Start with our comprehensive [**Contributor Onboarding Guide**](docs/ONBOARDING.md) for step-by-step environment setup, starter task ladders, and curated good first issues!
> 🖥 **Running the full stack locally?** See [**docs/LOCAL_STACK.md**](docs/LOCAL_STACK.md) for anvil, Stellar/Soroban, frontend, and web UI setup in one place.

## Ground rules

- **Do not start work on an issue until a maintainer assigns it to you. Comment on the issue to express interest.**
- Every PR must include `Closes #<issue>` in its description so Drips Wave can link it.
- Keep PRs to one issue. Do not change files outside the directories the issue covers.
- Report security problems privately — see [SECURITY.md](SECURITY.md).

---

## Setup

### Stellar / Soroban

1. **Rust.** Install [rustup](https://rustup.rs). The toolchain version, the `wasm32v1-none` target, `rustfmt`, and `clippy` are pinned in `stellar/rust-toolchain.toml`:

   ```bash
   cd stellar
   rustup toolchain install
   ```

2. **Stellar CLI** (version 28.x):

   ```bash
   cargo install --locked stellar-cli
   # or a prebuilt binary from https://github.com/stellar/stellar-cli/releases
   ```

   Check: `stellar --version`.

3. **Build and test**

   ```bash
   cd stellar
   cargo test
   stellar contract build   # writes target/wasm32v1-none/release/*.wasm
   ```

### Solidity / Foundry

1. **Foundry.** Install from <https://getfoundry.sh>. Pin matches CI: `1.2.3`.

2. **Install dependencies**

   ```bash
   git submodule update --init --recursive
   ```

3. **Build and test**

   ```bash
   forge build --sizes
   forge test -vvv
   ```

### Frontend

1. **Node.js** ≥ 20. Use [nvm](https://github.com/nvm-sh/nvm) or [fnm](https://github.com/Schniz/fnm) to manage versions.

2. **Install dependencies**

   ```bash
   cd frontend
   npm install
   ```

3. **Run the dev server**

   ```bash
   npm run dev
   ```

4. **Run checks**

   ```bash
   npm run lint
   npm run build   # ensure the production build succeeds
   ```

---

## Branch Strategy

Branch from `main` and name the branch by type and issue number:

| Type | Pattern | Example |
|---|---|---|
| Feature | `feat/<issue>-short-name` | `feat/42-staking-topups` |
| Bug fix | `fix/<issue>-short-name` | `fix/99-interest-rounding` |
| Documentation | `docs/<issue>-short-name` | `docs/180-stellar-api-reference` |
| Tests | `test/<issue>-short-name` | `test/105-token-fuzz` |

Keep branches short-lived. Rebase on `main` before opening a PR if the branch is behind.

---

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org):

```
feat(staking): allow topping up an existing stake
fix(lending): round shares down on deposit
docs(stellar): add error-code reference
test(token): cover burn_from
```

**Scopes:** `token`, `lending`, `staking`, `stellar` (workspace, CI, scripts), `docs`, `frontend`, `web`.

One logical change per commit. Squash fixups before requesting review.

---

## Pre-commit Hooks (optional, recommended)

Install [pre-commit](https://pre-commit.com) to catch formatting and lint issues locally:

```bash
pip install pre-commit   # or pipx/brew
pre-commit install
pre-commit run --all-files   # optional first run
```

Hooks are configured in `.pre-commit-config.yaml` (Foundry `forge fmt`, Stellar `cargo fmt`, Prettier, ESLint, and basic hygiene checks). Hooks that require tools you haven't installed are skipped automatically. See [DEVELOPMENT.md](DEVELOPMENT.md) for details.

---

## Before Opening a PR

### Stellar / Soroban checks

Run these from `stellar/`. CI runs the same commands.

```bash
cargo fmt --all --check
cargo clippy --all-targets --locked -- -D warnings
cargo test --locked
stellar contract build
```

### Solidity checks

```bash
forge fmt --check
forge build --sizes
forge test -vvv
```

### Frontend checks

```bash
npm run lint
npm run build
```

---

## Stellar-Specific Requirements

### Build / test / clippy

The Stellar CI job (`stellar.yml`) runs in order: `fmt → clippy → test → contract build`. Every step must pass. The toolchain and CLI version are pinned — do not change `stellar/rust-toolchain.toml` without a maintainer discussion.

### Contract code rules

- Every state-changing function that acts for a user calls `user.require_auth()`.
- Admin functions use the `#[only_owner]` macro.
- Errors go in the contract's `#[contracterror]` enum with an explicit numeric code. Use `panic_with_error!` to surface them. Do not `panic!` with strings.
- Use instance storage for configuration, persistent storage for per-user data, and always extend TTLs (use the `DAY_IN_LEDGERS` constants as the guide).
- Use `checked_*` math on `i128`. No floats, no unchecked arithmetic.
- Emit a `#[contractevent]` for every state change that users or indexers care about.
- Every public function has a `///` doc comment explaining its purpose, parameters, and any error conditions.

### Adding a new Stellar contract

1. Create a new crate under `stellar/contracts/<name>/`.
2. Add it to `stellar/Cargo.toml` workspace members.
3. Use workspace dependency versions (`soroban-sdk = { workspace = true }`).
4. Add unit tests in `src/test.rs` and integration scenarios in `stellar/tests/integration.rs`.

---

## Frontend Contribution Section

Changes to `frontend/` must:

- Follow the existing component structure and naming conventions.
- Not introduce new dependencies without a maintainer discussion.
- Pass `npm run lint` and `npm run build` with no errors.
- Include a brief description of the UI change in the PR body (a screenshot helps for visual changes).
- Not hardcode RPC endpoints, contract addresses, or API keys — use environment variables as defined in `.env.example`.

If a PR touches smart-contract-related UI (connecting a wallet, reading on-chain state, sending transactions), link to the relevant contract function in the PR description.

---

## Pull Requests

Fill in the PR template: include `Closes #<issue>`, a summary of what changed, how you tested it, and the checklist below. A maintainer will review your PR. Please respond to review comments within the Wave cycle.

## Code Review Checklist

Before marking your PR ready for review, confirm the following:

**All PRs**
- [ ] `Closes #<issue>` is in the PR description.
- [ ] The PR touches only files relevant to the issue.
- [ ] No secrets, keys, or tokens are committed.
- [ ] CI passes (fmt, clippy/lint, tests, build).

**Stellar / Soroban**
- [ ] `cargo fmt --all --check` passes.
- [ ] `cargo clippy --all-targets --locked -- -D warnings` passes.
- [ ] `cargo test --locked` passes.
- [ ] `stellar contract build` succeeds and produces `.wasm` files.
- [ ] All public functions have `///` doc comments.
- [ ] New error variants have explicit numeric codes in the `#[contracterror]` enum.
- [ ] Every state change emits an event.
- [ ] TTLs are extended for any new persistent storage keys.
- [ ] No `panic!` with strings; `panic_with_error!` only.

**Solidity**
- [ ] `forge fmt --check` passes.
- [ ] `forge build --sizes` passes.
- [ ] `forge test -vvv` passes.
- [ ] New functions and storage variables have NatSpec comments.

**Frontend**
- [ ] `npm run lint` passes.
- [ ] `npm run build` passes.
- [ ] No hardcoded addresses or keys.
- [ ] Screenshot or recording attached for visual changes.

**Documentation**
- [ ] All links resolve correctly.
- [ ] Code blocks are accurate against the current codebase.
- [ ] Spelling and grammar checked.
