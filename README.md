# ImpactSplit

**Charges a swap for the price impact that lasts, and gives back the impact that does not.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://impact-split.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/ImpactSplitHook.sol`](src/hooks/ImpactSplitHook.sol)
- **Licence:** Apache-2.0

## How it works

Market microstructure has separated two things for forty years that every AMM still treats as one. When a trade moves a price, part of that move is information, and the price stays where the trade put it. The rest is the cost of demanding liquidity right now, and the price comes back.

The first is called permanent impact and the second temporary, and the distinction is the whole reason a market maker can quote at all: they lose to the first and earn from the second. An AMM charges a fee on size and calls it done. Size is a bad proxy for either component.

A large trade in a deep pool moves nothing and pays the same rate as a small one that moves the price a long way, and a pool that raises its fee with volatility charges the informed and the uninformed identically because at the moment of the swap they are indistinguishable. They are indistinguishable at that moment. They are not indistinguishable later.

This hook charges every swap for the impact it causes, holds the charge, and then looks again after a settlement window: whatever share of the move has survived is paid to the liquidity providers who wore it, and whatever share has decayed is returned to the trader who was only ever renting liquidity. Nobody has to guess which kind of flow arrived. The price says so, afterwards, for free.

A trader who moves the price and is right pays. A trader who moves the price and is wrong is refunded and has paid only the pool's ordinary fee. That is the correct answer in both cases, and it is not reachable by any rule that has to decide at swap time.

## Prior art

Permanent and temporary impact are standard microstructure, from Kyle's lambda through Almgren-Chriss. On-chain, dynamic-fee hooks price volatility or realised spread at the moment of the swap, and markout-based fees (including this catalogue's own MarkoutFee) grade past flow to price the next trade. Deferring an individual swap's own charge, then splitting it between the providers and that same trader according to how much of its move survived a settlement window, is the contribution here.

## Where it does not help

A single swap's persistence is measured against whatever the price does next, including other people's flow, so per-trade it is noisy and only correct in expectation; a pool with very few trades per window will hand out refunds and charges that individually look arbitrary. The charge is escrowed in the swap's unspecified currency, so a trader collects refunds in whichever side their trades happened to leave, and an unsettled escrow earns nothing while it waits. Settlement is permissionless but not automatic, so an escrow nobody settles sits until somebody does. And the window is fixed at configuration: too short and everything looks permanent, too long and everything looks temporary.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    ImpactSplitHook.Config({
        maxImpactFee: /* uint24 */ 0,
        halfPointTicks: /* uint32 */ 0,
        windowSeconds: /* uint32 */ 0,
        minTicks: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `maxImpactFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `halfPointTicks` | `uint32` | ticks |
| `windowSeconds` | `uint32` | seconds |
| `minTicks` | `uint32` | ticks |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `HookFeeTooLarge()` | Fee is higher than the maximum allowed fee. |
| `InvalidConfig()` | A charge above the protocol's ceiling, or a window or half point of zero, is not a configuration. |
| `NoSuchEscrow()` | The escrow does not exist, or has already been settled. |
| `NothingToWithdraw()` | There is nothing to withdraw. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintDowncast(uint8,uint256)` | Value doesn't fit in a uint of `bits` size. |
| `TooSoon(uint256)` | The settlement window has not closed yet. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 4 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`
- `afterSwapReturnsDelta`

Mask: `0x10c4`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # ImpactSplit
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # fees, price-impact, microstructure, rebate, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/impact-split
cd impact-split
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
