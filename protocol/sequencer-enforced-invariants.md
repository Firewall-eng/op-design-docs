# Purpose

We ([Firewall](https://usefirewall.com/)) are building the Safe Sequencer for OP Stack chains. Safe sequencing uses exploit detection during block construction to exclude transactions from blocks that are identified as smart contract exploits. However, forced L1 → L2 messages can bypass the exploit detection of a safe sequencer and force include an exploit into a block.

# Summary

This is a design doc to enable (optional) sequencer-enforced invariants. We propose a design that uses a `forceReplay` toggle to save L1 -> L2 deposit messages in the `L2CrossDomainMessenger` rather than immediately executing them. This design allows the sequencer to control the ordering and inclusion of those messages with respect to it's own invariants that are outside of the rollup's core state-transition function.

We also include a `forceReplayController` that can specify the forced inclusion of arbitrary transactions with it's own logic when `forceReplay` is turned on, effectively bypassing the force replay behaviour in order to check the power of the sequencer.

# Problem Statement + Context

The OP Stack does not allow computation in the sequencer to determine deposit inclusion since L1 → L2 messages are always force executed on L2. This forced execution can bypass additional inclusion and ordering invariants offloaded to the sequencer role.

We believe delegating inclusion and ordering invariants to the sequencer can be a useful pattern, especially when heavy and complex computation is needed to enforce invariants, as seen in safe sequencing. This separation from the derivation pipeline and fault proof program of a rollup offers some nice benefits:

- Sequencer-enforced invariants can be added or removed from rollups through the SystemConfig.
- Sequencer-enforced invariants can be upgraded and managed at a pace appropriate for their own domain.
- Sequencer-enforced invariants cannot create bugs in the rollup’s fault proof program that effect withdrawals.

Sequencer-enforced invariants minimize complexity, yet still maintain rollup sovereignty.

# Proposed Solution

## Force Replay Toggle with Controller

A `forceReplay` setting is added to the `SystemConfig` contract. When `forceReplay` is turned on, deposit messages are saved in the `L2CrossDomainMessenger` rather than immediately executed. The sequencer can then choose to include or exclude the replay of these saved messages on L2. A `forceReplayController` contract can be set in the `SystemConfig` that allows for forced inclusion of arbitrary transactions with it's own logic when `forceReplay` is turned on.

### Forced Replay & Controller Design

Reference implementation: https://github.com/Firewall-eng/optimism

Two storage variables are added to the `SystemConfig` contract on L1: `FORCE_REPLAY` and `FORCE_REPLAY_CONTROLLER`. `FORCE_REPLAY` is a boolean that enables the `forceReplay` feature. `FORCE_REPLAY_CONTROLLER` is an address that points to the `forceReplayController` contract that implements the `IForceReplayController` interface.

When `forceReplay` is turned on, L1 -> L2 messages must come from the `L1CrossDomainMessenger` contract and target the `L2CrossDomainMessenger` contract or be a simple EOA to EOA ETH deposit. When the deposit message is executed on L2, the `L2CrossDomainMessenger` contract will check the `L1Block` contract to see if `forceReplay` is turned on. If it is, then the message is saved in the `L2CrossDomainMessenger` contract rather than executed.


### UX Considerations

When `forceReplay` is turned on, it is recommended that the sequencer immediately tries to insert a replay transaction for saved messages in the `L2CrossDomainMessenger` contract.

# Alternatives Considered

### Forced [no-op deposits](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md#proposed-solution) by the sequencer.

Unsafe payloads gossiped by the sequencer and data posted on L1 could be extended to include information that identifies certain transactions as failing the sequencer’s invariants. These transactions could be force-reverted, assuming a change to the execution layer. For deposit transactions, this could look a lot like the [no-op deposits specification](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md#proposed-solution). For normal transactions, this could charge a certain amount of gas used for built-in DDoS protection.

This requires modifying the derivation pipeline and does not allow deposit messages to be replayed unless transactions targeting the `L2CrossDomainMessenger` are handled specifically. These changes impact the fault proof program of a rollup and are more involved throughout the stack than the proposed solution. However, this solution still separates the actual computation for sequencer-enforced invariants from the rollup’s derivation and fault proof program. The result of the sequencer’s computation must be fed into the rollup’s derivation and fault proof program as an input.

### Embed “sequencer”-enforced invariants within the derivation pipeline and fault proof program.

In this case, we abandon the separation of concerns and move “sequencer”-enforced invariants and associated compute within the derivation and fault proof program of a rollup. In the case of safe sequencing, exploit detection algorithms must be run by each node on the network and embedded within the derivation and fault proof program of a rollup.

# Risks & Uncertainties

- Force replay should only be added to an existing network after a delay to allow users to exit if they don’t agree with the change. Should we embed this delay into the specification?

- We optimistically accept results shared by the sequencer and do not reorg the chain if the sequencer violates its promised invariants. This can make the design unworkable for modifications like [Interop](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md) where invariants must be true within the rollup’s state-transition function.

- In the proposed solution, the sequencer must manage DDoS attack vectors on its own, as it cannot charge gas for transactions that are not included in a block. The alternative solutions naturally solve the problem by charging gas used for transactions identified as failing a "sequencer"-enforced invariant.