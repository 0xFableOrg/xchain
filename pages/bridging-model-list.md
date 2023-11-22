# Bridging Model List

A list for existing and proposed [bridging
models](/pages/direction.md#bridging-models--user-experience). These models include cross-chain
messaging formats, as well as the relaying system. For instance, in the context of EVM third-party
bridges, the messaging format would mostly be the Solidity ABI of the contracts on both sides of the
bridge.

- [ERC-7533 - Public Cross Port](https://github.com/ethereum/ERCs/pull/62/files)
- [ERC-5164 - Cross-Chain Execution](https://eips.ethereum.org/EIPS/eip-5164)
  - Seems on the whole less complete than 7533.
- [Hyperlane](https://docs.hyperlane.xyz/)
- [IBC](https://github.com/cosmos/ibc)
- [Optimism Bridge](https://github.com/ethereum-optimism/optimism/tree/develop/specs)
  - In particular,
    [messengers](https://github.com/ethereum-optimism/optimism/blob/develop/specs/messengers.md),
    [bridges](https://github.com/ethereum-optimism/optimism/blob/develop/specs/bridges.md),
    [deposits](https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md) and
    [withdrawals](https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md).

## Non-Generic Models

Models that do not allow for generic message-passing.

- [ERC-7381 (xERC-20) Token Standard](https://github.com/ethereum/ERCs/pull/89/files)
- [Nexa Token Standard](https://nexanetwork.gitbook.io/nexa/)