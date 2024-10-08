# Milotruck Challenges Writeup

Challenges: https://github.com/MiloTruck/evm-ctf-challenges

For this set of challenges, each task has a `Setup.sol` to setup the environment (while DVD is doing this in each test's `setUp` function), to provide the player an onetime airdrop and to check the challenge solving status.

The author has already provided the solution in the `test` folder.
Since the test scripts have nothing to do with setting up and solution checks, I removed them and planned to write my own from scratch.

## GreyHats Dollar (2024/09/14)

This challenge defines a token(-like) contract with deflation.
There is shares, convertion rate, deflation rate.... All of them are not important (not related to the bug).

The contract allows transfering tokens to self.
This typically does not cause problems, however, when updating the balances(shares), the contract is not using inplace add/sub (i.e. `+=` or `-=`)
Instead, it first calculates the two new values, then sets them.
When transferring to self, the balances will be updated twice, and the second will overwrite the first.
In this case, it is the receiver's balance overwriting the payer's balance.
So, tokens emerged from nowhere will be added to our balance.

## Escrow (2024/09/15)

In this challenge, we need to drain an `Escrow`.

The `Escrow` has three non-view functions: `initialize` can only be used once, while `deposit` and `withdraw` are only for the owner.
The contract does not use `Ownable` or store the owner address, instead it queries its factory with its id (hash), and the factory records the owners using NFT.
Since the setup has renounced ownership of the `Escrow`, if we can deploy a new escrow with the same hash, we become the owner of both escrows.

The deployment of escrows is interesting.
The factory is using a library `ClonesWithImmutableArgs` (CWIA) to deploy (actually clone) the contract.
When calling functions of the deployed(cloned) contracts, it will internally append the "immutable args" to the calldata, and another `uint16` representing the arg length, then triggers a self-delegated call with the new calldata.
Then the contract can read the "immutable args" via `_getArgAddress`.

We cannot add other escrow implementations to directly return the hash, which is calculated using an identifier, the factory address and two token addresses.
So we have to deploy our escrow with same parameters.
However the factory rejects the copy-pasting of the deployment codes as it has already be deployed.

The immutable args (the two token addresses, and the second is `address(0)`) are provide in a combined `bytes`.
By appending other garbage chars, we can bypass the deployed check, but `initialize` will fail due to the overlong calldata.
If we directly drop the second zero address, the CWIA length will be read as non-zero address.
Since the length is less than 256, we can just drop a single null byte, and let the higher byte of CWIA length to fill it.

## Simple AMM Vault (2024/09/16)
There is an amm providing swap and flash loan service for SV and GREY.
SV is a vault, using GREY as its underlying token.
Its default price for SV:GREY is 1:1, while the setup has added reward to make the price 1:2.
The swap has an invariant `K`, calculating the balance in SV (GREY converted using the vault's price).
When swapping or loaning, the swap requires `K` not decreasing.

Notably, the amm is the only holder for SV tokens.
By flash loaning all the SV tokens from the swap, we can burn all of them (1000 SV) to withdraw all GREY in the vault (2000 GREY).
The price will revert to 1:1 as there is no balances, so again depositing 1000 GREY will mint 1000 SV for us to repay the loan.

Another problem lies at the check for the invariant.
The swap will calculate `K` only once after a swap or loan, and then compare it to the `K` when the last (de)allocate (adding or removing liquidity) occurrs.
By reverting the price, `K` actually raises beacuse GREY worth more SV now.
However, since the swap not updating `K`, we can withdraw some tokens (by swapping without payment or loaning without paying back) to reduce the actual `K` to the old level.

Alternatively, we can add liquidity with no token input, and the swap will still attribute the raise to us and mint LP tokens.

## Voting Vault (2024/09/18)

We need to withdraw from a treasury, which use the propose-vote-execute model.
It requires proposal and voting in different blocks, and reads the voting power at the time of the proposal.
We cannot change the past state, so everything in the treasury seems fine.

We need 1000000 votes to execute a proposal, but we only have 1300 votes (1000 Grey airdrop * 1.3 ratio).
There must be something suspicious in the voting vault.
Indeed, there are two `unchecked` actions.
If we can make the voting power underflow, we will hold almost infinite voting power.

The problem occurrs with the voting ratio. The vault calculates the votes as `amount * 1.3e18 / 1e18`.
If we lock 4 token separately, we only get 4 votes as each time the vaule is floored to 1.
Then we delegate the votes to someone else (cannot unlock due to the lock duration), our votes will be reduced by 5 - underflow!

## Meta Staking (2024/09/19)

Very similar to Damn Vulnerable Defi - Naive Receiver, refer [here](../A.damn-vulnerable-defi#naive-receiver-240830).

## Gnosis Unsafe (2024/09/20)

This bug is a compiler bug: https://soliditylang.org/blog/2022/08/08/calldata-tuple-reencoding-head-overflow-bug/

The argument layout of the functions matches the bug pattern. Though the blog states it occurs when forwarding the calldata to other contracts, it actually also happens in `abi.encode`.
Hence the signer field in the hashed data is overwritten to `0`, and we can queue a transaction with any owner and executing it with zero address signer.
When `ecrecover` fails, it returns zero address, so our transaction gets executed
