# VestedBuy

**Vests what a buyer receives, at the pool, so a launch can sell without handing every buyer a same-block exit.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://vested-buy.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/VestedBuyHook.sol`](src/hooks/VestedBuyHook.sol)
- **Licence:** Apache-2.0

## How it works

A token that wants its early buyers to hold has two options today and both are bad. Vesting in the token contract binds everyone forever, including the market makers and the exchanges the project needs, and it is the reason so many tokens ship with transfer restrictions nobody can later remove. Or the project vests nothing and watches the first hour decide the price, because a buyer who can sell immediately is not a holder, they are a position with a very short horizon.

The thing that actually wants vesting is not the token, it is the purchase. Buying through this pool during its vesting period delivers a share of the tokens now and the rest on a schedule, and the token itself is untouched: no transfer hooks, no allowlist, no permanent restriction, and every other venue trades it normally. A project can point its launch at this pool, get holders rather than flippers out of it, and still have a plain ERC-20.

The buyer keeps `immediateBps` of the fill and the hook holds the rest as ERC-6909 claims, releasing them linearly from `cliff` to `cliff + duration`. Claims are paid as real tokens. Attribution needs no signature.

`hookData` may name who the vested tokens belong to, and naming somebody else gives them your tokens, which is the only thing a forged attribution achieves. A buy that names nobody vests to the router it came through, which lets a launchpad run the vesting for its users. Exact-output buys are refused while vesting is live, and that is a security property rather than a limitation.

Uniswap v4 lets a hook adjust only the swap's unspecified currency, which on an exact-output buy is the input, not the output. A hook that quietly declined to vest those would be a hook whose vesting is bypassed by changing one field, so it says no instead.

## Prior art

Vesting is normally a property of the token (transfer restrictions, locked allocations) or of a distribution contract that holds an allocation and releases it. Vesting the purchase at the venue, so the token stays a plain ERC-20 and only buys through this pool are vested, is the contribution here.

## Where it does not help

It binds one pool. The same token bought anywhere else is not vested, so this shapes a launch rather than enforcing a lockup, and a project that needs the second should vest in the token. The schedule also cannot be cancelled or clawed back by anybody, including the project, which is deliberate but means a buyer who loses their key loses the unvested remainder.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    VestedBuyHook.Config({
        buyIsZeroForOne: /* bool */ 0,
        immediateBps: /* uint16 */ 0,
        cliff: /* uint32 */ 0,
        duration: /* uint32 */ 0,
        endsAt: /* uint64 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `buyIsZeroForOne` | `bool` |  |
| `immediateBps` | `uint16` | basis points (`10000` = 100%) |
| `cliff` | `uint32` |  |
| `duration` | `uint32` |  |
| `endsAt` | `uint64` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `EndsInThePast()` | A vesting period that has already ended cannot be configured. |
| `ExactOutputBuysRefused()` | Exact-output buys cannot be vested, so they are refused while vesting is live. See the contract notes. |
| `InvalidShare()` | `immediateBps` above 100% would deliver more than the buy produced. |
| `NothingVested()` | There is nothing unlocked to claim. |
| `PayoutNotPoolManager()` | Only the `PoolManager` may drive the callback. Named distinctly because `BaseHook` declares its own. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `afterSwap`
- `afterSwapReturnsDelta`

Mask: `0x1044`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # VestedBuy
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # launch, vesting, anti-flip, time, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/vested-buy
cd vested-buy
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
