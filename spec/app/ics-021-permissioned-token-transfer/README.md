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

This standard document specifies packet data structure, state machine handling logic for the transfer of permission information for [ICS 20](../ics-020-fungible-token-transfer/) token transfers. The state machine logic allows for an IBC denom on a non-native chain to be permissioned with respect to, who can send it, who can receive it and which IBC channel it can be transferred over. These permissions are controlled by the chain where the token was natively issued.

### Motivation

Users might wish to utilize a permissioned asset issued on one chain on another chain. An asset might be permissioned on the native chain, but when it is IBC transferred, the asset issuer loses their capability to enforce their permissions. This application-layer standard describes a protocol for communication of asset permissions between chains connected with IBC. The permissions need to be updated only on the native chain which will then be propogated over to all the relevant IBC connected chains.

### Definitions

- `Permissioned Token`: A token which might be a natively issued on a chain or created as a voucher from an ICS20 transfer, which is permissioned by an Owner on the native chain.
- `Owner`: The account which sets the permissions for a Permissioned Token. This will likely be the creator of the token.
- `Host Chain`: The chain where the permissioned tokens are considered native. The host chain facilitates connections to mirror chains, and ensures the propagation of token specific permissions.
- `Host ICS20 Channel`: The channel on the Host Chain which is connected to the mirror chain using the ICS20 protocol and is used to transfer funds to and from the Host Chain to the Mirror Chain.
- `Mirror Chain`: The chain receiving the permissioned tokens and issuing *controlled* voucher tokens. It is up to the mirror chain to enforce the propagated permissions.
- `Mirror ICS20 Channel`: The channel on the Mirror Chain which is connected to the Host Chain usinng the ICS20 protocol and is used to transfer funds to and from the Mirror Chain to the Host Chain.
- `AccountBlocklist`: A group of publickeys that aren't allowed to interact with a specific permissioned token. 
- `ChannelAllowlist`: A list of ICS20 channels the permissioned token can be sent across. 

### Desired Properties

- Preservation of account permissions crosschain, which can forbid an account from 
  1. Sending the permissioned token
  2. Receiving the permissioned token
  3. Using the permmissioned token to pay for gas // todo ? is this needed?
  4. Transferring the token back to Host Chain // todo ? should this be disallowed? Should even a blacklisted user be always able to send tokens to themselves on noble? and funds locked there

- Preservation of transfer permissions crosschain which allows transfer only from
  1. Host Chain to Mirror Chain using a channel in ChanneAllowlist
  2. Mirror Chain to Host Chain across the channel it came from

- Application of the permissions to tokens which have already been transferred before the configuration of ICS21 e.g multiple ics20 channels created and amount transferred before the creation of ICS21 channel. these should be addressed or at least identified

## Technical Specification

(main part of standard document - not all subsections are required)

(detailed technical specification: syntax, semantics, sub-protocols, algorithms, data structures, etc)

### General Design

The Host Chain is responsible for hosting a native token issuance mechanism. Any issued token denom should expose an Owner which controls the token issuance. Any number of ICS20 channels can be created between Host Chain and Mirror Chain for the transfer of these tokens. When the Owner decides to make the token permissioned, they register it with the ICS21 Host Module.  

Once a Permissioned Token has been registered on the Host Chain, the Owner can now set the persmissions customization. This would include:
1.  The ICS20 Channel IDs over which the Permissioned Token denom can be sent or received. // todo: is there any need to do receive check? should send check be enough? this will solve the potential issue in the 2nd paragraph below

2. The list of pubkeys which are not allowed to interact with the Permissioned Token on Host Chain or the Mirror Chain. The pubkeys are used instead of account address as this allows the Mirror Module to derive the addresses from the pubkeys when it is locally storing the permissions. The Host Chain need not care about the address encoding schemes.

The scope of the ICS21 channel is limited to communicating the upto-date Permissions of the token. The transfer of the same happens via the permissionless ICS20 protocol. 
This causes some gotcha situations where an ICS20 channel might exist over which the Permission Token has been sent before it was registered as a Permissioned Token on the Host Chain. If that Channel ID is not part of the configured ChannelAllowlist, those tokens are now stuck on that channel with no way to retreive them back to the Host Chain.

The native transfer (from account to account on same chain) of a Permissioned Token is implemented via the chain's SendRestrictions exposed by the Cosmos-SDK x/bank module. On every transfer, the function which implements the ICS21 SendRestrictions will check if the transfer token denom belongs to a known ICS21 Permissioned Token. If so, the sender and receiver addresses are checked against the AccountBlocklist. If either are in the AccountBlocklist, the transfer is cancelled.  

### Data Structures

We propose a new Packet structure to be used to communicate the tokens latest permissions from the Host Chain to the Mirror Chain, when they are updated by the Owner.

```typescript
interface SetAccountBlocklistPacket {
  // denom is the Host Chain denom of the Permissioned Token
  denom: string
  // accountblocklist_additions is the list of pubkeys added to the blocklist with
  // the new permissions update by the Owner 
  accountblocklist_additions: bytes[]
  // accountblocklist_removals is the list of pubkeys removed from the blocklist with
  // the new permissions update by the Owner
  accountblocklist_removals: string[]
}
```

There is no need for custom Acknowledgement packet as the Mirror Chain does not send anything useful back to the Host Chain.

### Sub-protocols

(sub-protocols, if applicable)

### Port & channel setup

An ICS 21 Host module must always bind to a port with the id `ics21host`. Mirror Chains will bind to port `ics21mirror`.

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

### Host Chain Contract

### New Token FLow 

### Update Permission Flow

### Cosmos-SDK Contract

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

## Future Improvements

Handle trahsfer natively in the protocol insteaed of relying on ICS20 and building wrappers around it

## History

(changelog and notable inspirations / references)

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
