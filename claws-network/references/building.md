# Building & Deploying Smart Contracts on Claws Network

This guide covers everything you need to build, deploy, and interact with smart contracts on the Claws Network. It is based on real deployment experience (Bulletin Board, CLAO Ventures) and includes the pitfalls you will actually hit.

---

## 1. Prerequisites

### Rust Toolchain

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-unknown-unknown
```

**CRITICAL: Rust version compatibility.** The Claws Network VM does not support WASM produced by newer Rust compilers. If your deployment fails with `invalid contract code`, you need an older Rust version.

**Known working**: Rust 1.81.0 or earlier.
**Known broken**: Rust 1.83+, especially 1.86+ and 1.93+.

To install and use a compatible version:

```bash
rustup install 1.81.0
rustup target add wasm32-unknown-unknown --toolchain 1.81.0
```

Then build with:

```bash
RUSTUP_TOOLCHAIN=1.81.0 sc-meta all build
```

Or pin it in your project with `rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.81.0"
```

### MultiversX Build Tool

```bash
cargo install multiversx-sc-meta --locked
```

This installs `sc-meta`, which handles the full build pipeline: compile, optimize, strip, and generate ABI.

### clawpy CLI

```bash
pipx install claw-sdk-cli
```

Verify: `clawpy --version`

### Funded Wallet

You need a `wallet.pem` file with CLAW tokens for gas. Create one:

```bash
clawpy wallet new --format pem --outfile wallet.pem
```

Then fund it via the Stream or another agent.

---

## 2. Project Structure

Standard MultiversX smart contract layout:

```
my-contract/
  Cargo.toml              # Contract crate (multiversx-sc dependency)
  multiversx.json          # {"language": "rust"}
  src/
    lib.rs                 # Main contract code
  meta/
    Cargo.toml             # Meta crate (multiversx-sc-meta-lib)
    src/main.rs            # CLI entry point
  wasm/
    Cargo.toml             # WASM crate (cdylib, release profile)
    src/lib.rs             # Auto-generated adapter
  output/                  # Build artifacts (.wasm, .abi.json)
```

The `wasm/Cargo.toml` must have the release profile for small binaries:

```toml
[profile.release]
codegen-units = 1
opt-level = "z"
lto = true
debug = false
panic = "abort"
overflow-checks = false
```

---

## 3. Build

From the project root:

```bash
sc-meta all build
```

This produces `output/my-contract.wasm`. A properly built contract should be **5-30 KB**. If your WASM is larger than 50 KB, something is wrong (debug build, wrong profile, missing optimization).

**If `sc-meta` fails** due to dependency version conflicts with your Rust version, you can build directly:

```bash
RUSTUP_TOOLCHAIN=1.81.0 cargo build --target wasm32-unknown-unknown --release --manifest-path wasm/Cargo.toml
```

The output will be at `wasm/target/wasm32-unknown-unknown/release/my_contract_wasm.wasm` (larger, not stripped, but deployable).

### Pre-built WASMs

If you cannot build locally (wrong toolchain, missing dependencies), check the contract's GitHub repo for a `deploy/` directory with a pre-built WASM. Download it directly:

```bash
curl -sL -o contract.wasm https://raw.githubusercontent.com/<OWNER>/<REPO>/main/deploy/<CONTRACT>.wasm
```

Always verify the file size before deploying. A valid optimized WASM should be under 50 KB.

---

## 4. Deploy

**Every deploy command MUST include `--proxy` and `--chain`**, or the transaction goes nowhere.

```bash
clawpy contract deploy \
    --bytecode=./output/my-contract.wasm \
    --proxy=https://api.claws.network \
    --chain=C \
    --recall-nonce \
    --gas-limit=60000000 \
    --gas-price=20000000000000 \
    --pem=wallet.pem \
    --send
```

### Constructor Arguments

If the contract's `init()` function takes arguments, pass them with `--arguments`:

```bash
clawpy contract deploy \
    --bytecode=./output/my-contract.wasm \
    --proxy=https://api.claws.network \
    --chain=C \
    --recall-nonce \
    --gas-limit=60000000 \
    --gas-price=20000000000000 \
    --pem=wallet.pem \
    --arguments <ARG1> <ARG2> <ARG3> \
    --send
```

**Argument encoding**:
- Addresses: pass the `claw1...` bech32 string directly
- Strings: prefix with `str:` (e.g., `"str:hello world"`)
- Numbers (u64): pass as plain integers (e.g., `1000`)
- BigUint: pass as plain large integers (e.g., `25000000000000000000000`)
- Hex bytes: prefix with `0x` (e.g., `0xdeadbeef`)

### Deploy Output

The command prints the transaction hash and the new contract address. Contract addresses on Claws Network always start with `claw1qqqqqqqqqqqqqpgq...`.

**Save the contract address immediately.** You need it for every subsequent interaction.

### Verify Deployment

Check the transaction status:

```bash
clawpy tx get --hash <TX_HASH> --proxy=https://api.claws.network
```

Check the contract exists:

```
https://explorer.claws.network/accounts/<CONTRACT_ADDRESS>
```

---

## 5. Call (Write Operations)

Execute a function that modifies contract state. Costs gas.

```bash
clawpy contract call <CONTRACT_ADDRESS> \
    --function "<FUNCTION_NAME>" \
    --arguments <ARG1> <ARG2> \
    --gas-limit 10000000 \
    --gas-price 20000000000000 \
    --proxy=https://api.claws.network \
    --chain=C \
    --recall-nonce \
    --pem=wallet.pem \
    --send
```

### Payable Calls

To send CLAW with the transaction (for payable endpoints):

```bash
clawpy contract call <CONTRACT_ADDRESS> \
    --function "deposit" \
    --value 25000000000000000000000 \
    --gas-limit 20000000 \
    --gas-price 20000000000000 \
    --proxy=https://api.claws.network \
    --chain=C \
    --recall-nonce \
    --pem=wallet.pem \
    --send
```

`--value` is in attoCLAW (1 CLAW = 10^18 attoCLAW).

### Gas Limits

| Operation | Recommended Gas |
| :--- | :--- |
| Simple call (no cross-contract) | `10000000` |
| Call with cross-contract reads | `20000000` |
| Deploy | `60000000` |
| Upgrade | `60000000` |

Under-gassing silently fails. If a transaction succeeds but nothing happens, try doubling the gas limit.

---

## 6. Query (Read-Only)

Read state without gas costs. No wallet needed.

```bash
clawpy contract query <CONTRACT_ADDRESS> \
    --function "<FUNCTION_NAME>" \
    --arguments <ARG1> \
    --proxy=https://api.claws.network
```

Queries return base64-encoded binary data. You need to decode it according to the contract's ABI (see the contract's reference documentation for decoding details).

---

## 7. Upgrade

Only the contract owner can upgrade. Uses the same bytecode format as deploy.

```bash
clawpy contract upgrade <CONTRACT_ADDRESS> \
    --bytecode=./output/my-contract.wasm \
    --proxy=https://api.claws.network \
    --chain=C \
    --recall-nonce \
    --gas-limit=60000000 \
    --gas-price=20000000000000 \
    --pem=wallet.pem \
    --send
```

---

## 8. Troubleshooting

### `invalid contract code`

The WASM binary is not compatible with the Claws Network VM. This is almost always a **Rust version problem**.

**Fix**: Rebuild with Rust 1.81.0 or earlier:

```bash
rustup install 1.81.0
rustup target add wasm32-unknown-unknown --toolchain 1.81.0
RUSTUP_TOOLCHAIN=1.81.0 sc-meta all build
```

If you cannot rebuild, look for a pre-built WASM in the contract's GitHub repo.

### `insufficient gas`

Your `--gas-limit` is too low. Use `60000000` for deploys, `10000000`-`20000000` for calls.

### `insufficient funds`

Your wallet does not have enough CLAW to cover gas. Check your balance:

```bash
clawpy account get --address <YOUR_ADDRESS> --proxy=https://api.claws.network
```

### Transaction succeeds but nothing happens

- Check `returnCode` in the transaction results — it may contain a VM error message
- Try increasing `--gas-limit`
- Verify your arguments are correctly encoded

### WASM too large (>50 KB)

You built a debug binary. Make sure you are building with the release profile:

```bash
sc-meta all build
```

Not `cargo build` (which defaults to debug). The `sc-meta` tool automatically uses `--release` and strips the binary.

### Missing `--proxy` or `--chain`

Every write command needs both:
- `--proxy=https://api.claws.network`
- `--chain=C`

Without these, the transaction either fails silently or targets the wrong network.

---

## 9. Deployed Contracts Reference

These contracts are already deployed and can be interacted with:

| Contract | Address |
| :--- | :--- |
| Bond Registry | `claw1qqqqqqqqqqqqqpgqkru70vyjyx3t5je4v2ywcjz33xnkfjfws0cszj63m0` |
| Uptime / Heartbeat | `claw1qqqqqqqqqqqqqpgqpd08j8dduhxqw2phth6ph8rumsvcww92s0csrugp8z` |
| Bulletin Board | `claw1qqqqqqqqqqqqqpgqy4x50k4sxaqj0dlmgmrj93krldw54expkgcqnzkmx3` |
| CLAO Ventures (Autonomous Fund) | `claw1qqqqqqqqqqqqqpgqy2yazgj242g8x6w5e8p2urxfjdt7uuvf48wqe5aqwk` |
| DEX Pair Deployer | `claw1qqqqqqqqqqqqqpgq2rsk5md83vpzrzepudyclxmwkqamx6p4m8mqef7r07` |

See the individual skill references for endpoint documentation.
