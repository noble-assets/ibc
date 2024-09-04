---
ics: 21
title: Permissioned Token Transfer
stage: draft
category: IBC/APP
kind: instantiation
author: John Letey <john@nobleassets.xyz>, Daniel Kanefsky <dan@nobleassets.xyz>
created: 2024-06-14
modified: 2024-06-14
requires: 25, 26
required-by: (optional list of ics numbers)
implements: (optional list of ics numbers)
version compatibility: (optional list of compatible implementations' releases)
---

> This standard document follows the same design principles of [ICS 20](../ics-020-fungible-token-transfer) and inherits most of its content therefrom.

## Synopsis

(high-level description of and rationale for specification)

### Motivation

(rationale for existence of standard)

### Definitions

- `Host Chain`: The chain where the permissioned tokens are considered native. The host chain facilitates connections to mirror chains, and ensures the propagation of token specific allowlists and blocklists.
- `Mirror Chain`: The chain receiving the permissioned tokens and issuing *controlled* voucher tokens. It is up to the mirror chain to enforce the propagated allowlists and blocklists.
- `Allowlist`: A group of addresses that are allowed to interact with a permissioned token. Any address not on the allowlist is forbidden to interact with the token.
- `Blocklist`: A group of addresses that aren't allowed to interact with a permissioned token. Any address not on the blocklist is allowed to interact with the token.

The IBC handler interface & IBC routing module interface are as defined in [ICS 25](../../core/ics-025-handler-interface) and [ICS 26](../../core/ics-026-routing-module), respectively.

### Desired Properties

(desired characteristics / properties of protocol, effects if properties are violated)

## Technical Specification

(main part of standard document - not all subsections are required)

(detailed technical specification: syntax, semantics, sub-protocols, algorithms, data structures, etc)

### Data Structures

We utilize the existing [ICS 20 `FungibleTokenPacketData`](../ics-020-fungible-token-transfer/README.md#data-structures) structure to transfer tokens over ICS 21 channels in order is to maintain client compatibility. Note that the Host Chain will block all transfers of permissioned token transfers over non-ICS 21 channels.

Additionally, we define a new packet data type for the propagation of token allowlists and blacklists.

```typescript
interface PermissionPropagationPacketData {
  denom: string
  allowlist_additions: string[]
  allowlist_removals: string[]
  blocklist_additions: string[]
  blocklist_removals: string[]
}
```

### Sub-protocols

(sub-protocols, if applicable)

### Port & channel setup

An ICS 21 Host module must always bind to a port with the id `ics21host`. Mirror Chains will bind to ports dynamically, as specified in the identifier format [section](#identifier-formats).

The example below assumes a module is implementing the entire `ICS21HostModule` interface. The `setup` function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port.

```typescript
function setup() {
  capability = routingModule.bindPort("ics21host", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onChanUpgradeInit, // read-only
    onChanUpgradeTry,  // read-only
    onChanUpgradeAck,  // read-only
    onChanUpgradeOpen,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
  claimCapability("port", capability)
}
```

Once the `setup` function has been called, channels can be created via the IBC routing module.

### Channel Lifecycle Management

An ICS21 Channel can only be initialized by the Host module. As the token is issued on the host chain, and the transfer permissions are set from the host chain, the channel too must be initialized from the host chain. `onChanOpenInit` on the mirror chain should return error.

```typescript
// Called on Host Chain by Relayer
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string
): (version: string, err: Error) {
  abortTransactionUnless(portIdentifier === "ics21host")
  // only allow channels to be created on the "ics21mirror" port on the counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "ics21mirror") 
  // currently only v1 of ICS21 is supported
  abortTransactionUnless(version === "ics21-1")

  return version, nil
}
```
Since the channel is always initialized by the host module, `onChanOpenTry` on the host module should return an error. The mirror module would continue with the handshake.

```typescript
// Called on Mirror Chain by Relayer
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string
): (version: string, err: Error) {
  abortTransactionUnless(portIdentifier === "ics21mirror")
  // only allows channels to be created from the "ics21host" on the couterparty chain
  abortTransactionUnless(counterpartyChannelIdentifier === "ics21host")
  // ensure that the host module is running on the same version we expect
  abortTransactionUnless(counterpartyVersion === "ics21-1")
  
  mirrorModuleVersion = "ics21-1"
  return mirrorModuleVersion, nil
}
```

`onChanOpenAck` on mirror chain should return an error.

```typescript
// Called on Host chain by Relayer
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyChannelIdentifier,
  counterpartyVersion: string
) {
  // ensure that the mirror module is running on the same version as we expect
  abortTransactionUnless(counterpartyVersion === "ics21-1")
}
```
`onChanOpenConfirm` on host chain should return an error.

```typescript
// Called on Mirror Chain by Relayer
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier
) {
  // no-op
}
```

### Closing handshake

// todo who can initiate channel close? just host? should mirror be able to exit as well (probably not). why would hosts want to close? should we have conditions that all AllowedChannels store should be empty (wrt channels on that chain) to be able to close. If not, Should we only allow the close of the channel if there are no ics20tokens of denom across the channel?
```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
 	// todo
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    // todo
}
```

### Upgarde Handshake

// todo : not needed for now imo, but when we wanna do ics21-2? which might have native trasfer functionality
Advantages of native transfer, easy to ensure that tokens arent trasferred anywhere else. they all stay in the same ics21-2 channel and can only be sent back to host.  no multi hops.

### Packet relay

`onRecvPacket` need not be implemented for the Host chain as the Mirror chain does not issue any packets. The mirror chain on `onRecvPacket` stores the new and updated channel data.

```typescript
// Called on Mirror Chain by Relayer
function onRecvPacket(packet Packet) {
  ack = NewResultAcknowledgement([]byte{byte(1)})

  var data: ics21types.SetAllowedChannelPacket
  data, err = packet.GetData() 
  if err != nil {
    return NewErrorAcknowledgement(ics21types.ErrInvalidType)
  }

  if data.Signer != authtypes.NewModuleAddress(ics21types.ModuleName+data.CounterpartyChannelId) {
    return NewErrorAcknowledgement(ics21types.ErrInvalidSigner)
  }

  err = StoreAllowedChannels(data)
  if err != nil {
    return NewErrorAcknowledgement(err)
  }
  return NewAcknowledgement(result)
}
```

`onAcknowledgePacket` need not be implemented for the Mirror chain as it does not issue any packets. The host chain on `onAcknowledgePacket` checks it any error was found on the counterparty chain and resets its state to reflect the Mirror state.

```typescript
// Called on Host Chain by Relayer
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes
) {
  var data: ics21types.SetAllowedChannelPacket
  data, err = packet.GetData() 
  if err != nil {
    return NewErrorAcknowledgement(ics21types.ErrInvalidType)
  }

  switch typeof(acknowledgement) {
    case *channeltypes.Acknowledgement_Error:
        RemoveAllowedChannel(data)
    default:
      // todo: are there any other potential errors we need to address? idts but verify
  }
}
```

`onTimeoutPacket` need not be implemented on Mirror chain. On Host, in case a packet timesout, that would mean the counterparty never got the state update, and as such the local state should be updated to reflect that.

```typescript
// Called on Host Chain by Relayer
function onTimeoutPacket(packet: Packet) {
  var data: ics21types.SetAllowedChannelPacket
  data, err = packet.GetData() 
  if err != nil {
    return NewErrorAcknowledgement(ics21types.ErrInvalidType)
  }
  RemoveAllowedChannel(data)
}
```


### Identifier formats

Host Port Identifier: `ics21host`
Mirror Port Identifier: `ics21mirror`

### Properties & Invariants

(properties & invariants maintained by the protocols specified, if applicable)

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

This initial standard uses version `"ics21-1"` in the channel handshake.

A future version of this standard could use a different version in the channel handshake, and safely alter the packet data format & packet handler semantics.

## Example Implementations

- An implementation of ICS 21 Host & Mirror in Golang can be found [here](https://github.com/noble-assets/ics21).
- An implementation of ICS 21 Mirror in [CosmWasm](https://cosmwasm.com) can be found [here](https://github.com/noble-assets/cw-ics21).

## History

(changelog and notable inspirations / references)

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
