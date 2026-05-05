# How To: Claim Old Rollup Sequencer Rewards via Etherscan

The Aztec staking dashboard at https://stake.aztec.network/ does not surface unclaimed
sequencer rewards on the old rollup. They are still claimable on-chain — you just have
to drive the contracts directly. This doc walks through doing it from Etherscan.

It's written for a **solo staker** claiming their own rewards. Providers running split
contracts have one extra step at the end; see [Provider addendum: distributing from a
PullSplit coinbase](#provider-addendum-distributing-from-a-pullsplit-coinbase).

## Background

After the alpha upgrade, validators migrated to the new rollup. Rewards earned on the
**old** rollup are still owed to each validator and need to be claimed manually. A
single permissionless call — `claimSequencerRewards(attester)` on the old rollup —
pulls the validator's pending AZTEC out of the rollup and sends it to whatever
**coinbase** address the validator was configured with.

For most solo stakers, the coinbase is just your own EOA, so the claim alone is
enough — the AZTEC lands directly in your wallet.

The call is permissionless: any wallet can trigger it. You just need ETH for gas. The
AZTEC always flows to the configured coinbase, never to the caller.

## Key addresses

| Item | Address |
|------|---------|
| Old rollup contract | `0x603bb2c05d474794ea97805e8de69bccfb3bca12` |
| AZTEC token | `0xa27ec0006e59f245217ff08cd52a7e8b169e62d2` |

For each validator you claim for you'll need its **attester** address (the validator's
signing/attestation address — not the coinbase, not the withdrawal address).

## Prerequisites

- A wallet with a small amount of ETH for gas (MetaMask, Rabby, Frame, Ledger Live, etc.).
  Each claim costs roughly the gas of a token transfer. Budget ~0.001 ETH per validator
  at moderate gas prices.
- The attester address(es) you intend to claim for.

## Step 1 — (Optional) Check pending rewards before paying gas

Use the rollup's read interface to confirm there is something to claim.

1. Open the old rollup on Etherscan:
   https://etherscan.io/address/0x603bb2c05d474794ea97805e8de69bccfb3bca12
2. Click **Contract** → **Read Contract** (proxy reads may live under **Read as Proxy**;
   try both if the function isn't visible).
3. Find `getSequencerRewards(address)` and paste the **attester** address.
4. Click **Query**. Result is in wei (18 decimals). Divide by `10^18` to get AZTEC.

If the value is `0`, there is nothing to claim for that attester — skip it.

## Step 2 — Claim rewards from the old rollup

1. Same Etherscan page → **Contract** → **Write Contract** (or **Write as Proxy** if
   the function is only visible there).
2. Click **Connect to Web3** and connect your wallet. Make sure the wallet is on
   **Ethereum mainnet**.
3. Locate `claimSequencerRewards`. It takes a single argument:
   - `_sequencer` (address) — the **attester** address (not the coinbase address).
4. Paste the attester address and click **Write**.
5. Confirm in your wallet. Wait for the transaction to mine.

Repeat for each attester. The AZTEC now sits at the configured coinbase address — your
wallet, for the typical solo-staker setup.

## Step 3 — Verify

- On Etherscan, view the AZTEC token (`0xa27ec0006e59f245217ff08cd52a7e8b169e62d2`)
  and check the `Token Holdings` of your coinbase address — it should reflect the
  newly received AZTEC. You can also check the token's transfer history filtered to
  your address.
- Re-run `getSequencerRewards(attester)` from Step 1 — it should now return `0`.

If your coinbase is not your own EOA but a PullSplit contract (provider setup), the
AZTEC is now in the split contract and one more step is needed — see the addendum
below.

## Common gotchas

- **Claiming on the wrong rollup.** Make sure you're on the **old** rollup
  (`0x603bb2c05d474794ea97805e8de69bccfb3bca12`). The new rollup has its own pending
  rewards and a separate `claimSequencerRewards` state.
- **Passing the coinbase or withdrawal address instead of the attester.**
  `claimSequencerRewards` takes the **attester** address.
- **Read vs. Write as Proxy.** The rollup is a proxy; functions may only show up under
  the "as Proxy" tabs. Try both if a function name is missing.
- **Gas.** `claimSequencerRewards` can occasionally need a higher manual gas limit.
  Etherscan's auto-estimate is usually fine, but if a tx reverts with out-of-gas, set a
  manual limit in your wallet before signing (200,000 is a safe starting point).
- **Coinbase set to the attester EOA.** In some early/misconfigured setups the coinbase
  is the attester's own address. The claim still works, but the AZTEC lands in the
  attester EOA — to move it elsewhere you need to sign a token `transfer` with the
  attester's private key. Etherscan can't do that for you; use a wallet that holds the
  key, or a script that signs locally.

---

## Provider addendum: distributing from a PullSplit coinbase

If you're a provider and your coinbase is a 0xSplits v2 **PullSplit** (commonly used to
share rewards across operators / fee recipients), Step 2 lands AZTEC in the split
contract — not in any recipient's wallet. PullSplit does not auto-forward; you have to
call `distribute(...)` on it. This call is also permissionless (some splits offer a
small caller incentive).

You need the split's parameters: recipient list, allocations, total allocation, and
distribution incentive. These were set when the coinbase was created. Sources, in order
of preference:

- **0xSplits app** — https://app.splits.org/accounts/&lt;split-address&gt;/?chainId=1
  shows the recipients, allocations, and a one-click "Distribute" button.
- **The split's creation transaction** on Etherscan — decoded `createSplit` input has
  the same data.
- **On-chain reads** on the split itself if exposed.

### Doing it on Etherscan directly

1. Open the **coinbase (PullSplit)** address on Etherscan and go to **Contract** →
   **Write Contract** (Connect Web3 first).
2. Find the `distribute` function. The PullSplit v2 signature is:
   ```
   distribute(
     SplitV2Lib.Split _split,    // tuple: (address[] recipients, uint256[] allocations, uint256 totalAllocation, uint16 distributionIncentive)
     address _token,             // 0xa27ec0006e59f245217ff08cd52a7e8b169e62d2 (AZTEC)
     address _distributor        // who gets the distribution incentive (your address, or zero)
   )
   ```
   Some deployments expose a second overload that also takes the current balance — if
   you see two `distribute` entries, use the one matching the args above.
3. Fill in `_split` as a tuple in Etherscan's UI (Etherscan accepts JSON-style tuples,
   e.g. `[["0xrecipient1","0xrecipient2"], [500000,500000], 1000000, 0]`).
4. `_token` = the AZTEC token address above.
5. `_distributor` = your wallet (or `0x0000000000000000000000000000000000000000` to
   forgo any incentive).
6. **Write** and confirm. After confirmation, AZTEC will be sitting in each recipient's
   wallet per the split's allocations.

### Easier path — 0xSplits app

If you'd rather not hand-encode a tuple in Etherscan, visit
https://app.splits.org/accounts/&lt;split-address&gt;/?chainId=1, connect the same
wallet, pick the AZTEC token row, and click **Distribute**. The app encodes the tuple
for you and submits the same call.

### Verify the distribute

- Each recipient address should show an inbound AZTEC transfer for their share.
- The PullSplit address's AZTEC balance should be ~0 (some dust may remain due to
  integer division of allocations).
