---
ics: 21
title: Permissioned Token Transfer
stage: draft
category: IBC/APP
kind: instantiation
author: John Letey <john@nobleassets.xyz>, Daniel Kanefsky <dan@nobleassets.xyz>
created: 2024-06-14
modified: 2024-06-14
requires: 20, 25, 26
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

- Preservation of transfer permissions crosschain which allows transfer only from
  1. Host Chain to Mirror Chain using a channel in ChanneAllowlist
  2. Mirror Chain to Host Chain across the channel it came from

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
  // denom is the Mirror Chain IBC denom of the Permissioned Token
  denom: string
  // accountblocklist_additions is the list of pubkeys added to the blocklist with
  // the new permissions update by the Owner 
  accountblocklist_additions: bytes[]
  // accountblocklist_removals is the list of pubkeys removed from the blocklist with
  // the new permissions update by the Owner
  accountblocklist_removals: bytes[]
}
```

There is no need for custom Acknowledgement packet as the Mirror Chain does not send anything useful back to the Host Chain.

### Sub-protocols

The sub-protocols described herein should be implemented in Host and Mirror modules with access to the bank module and to the IBC modules.

#### Port Setup

##### ICS21 Host

An ICS 21 Host module must always bind to a port with the id `ics21host`. 

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

##### ICS21 Mirror

An ICS 21 Host module must always bind to a port with the id `ics21mirror`. 

The example below assumes a module is implementing the entire `ICS21MirrorModule` interface. The `setup` function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port.

```typescript
function setup() {
  capability = routingModule.bindPort("ics21mirror", ModuleCallbacks{
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

#### Channel Setup Flow

An ICS21 Channel can only be initialized by the Host module. As the token is issued on the Host Chain, and the transfer permissions are set from the host chain, the channel too must be initialized from the host chain. `onChanOpenInit` on the mirror chain should return error.

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
  // only unordered channels allowed. this is bcuz ordered channels close on timeout
  // we dont want that. do we? todo
  // but also, because we only submit the changeset of permissions and not the entire 
  // permission set, Ordered channels would make sense, to ensure n packet is accepted 
  // before n+1. e.g we  add Alice to bloccklist in n, but remove Alice in n+1. if they
  // are processed in wrong order, we will end up with state where Alice is still blocked
  abortTransactionUnless(order === UNORDERED)
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

#### Channel Closing Flow

// todo who can initiate channel close? just host? should mirror be able to exit as well (probably not). why would hosts want to close? should we have conditions that all AllowedChannels store should be empty (wrt channels on that chain) to be able to close. If not, Should we only allow the close of the channel if there are no ics20tokens of denom across the channel? Is this entrypoint hit when an ordered channels is closed due to timeout? will need to address channel closure due to client expiry anyway

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

#### Upgarde Handshake

// todo : not needed for now imo, but when we wanna do ics21-2? which might have native transfer functionality. Advantages of native transfer, easy to ensure that tokens arent trasferred anywhere else. they all stay in the same ics21-2 channel and can only be sent back to host. no multi hops. 

#### Packet relay

`onRecvPacket` need not be implemented for the Host chain as the Mirror chain does not issue any packets. The mirror chain on `onRecvPacket` stores the new and updated channel data.

```typescript
// Called on Mirror Chain by Relayer
function onRecvPacket(packet Packet) {
  ack = NewResultAcknowledgement([]byte{byte(1)})

  var data: ics21types.SetAccountBlocklistPacket
  data, err = packet.GetData() 
  if err != nil {
    return NewErrorAcknowledgement(ics21types.ErrInvalidType)
  }

  if data.Signer != authtypes.NewModuleAddress(ics21types.ModuleName+data.CounterpartyChannelId) {
    return NewErrorAcknowledgement(ics21types.ErrInvalidSigner)
  }
/// todo: curent implementation also shares the whitelisted channel IDs to mirror. do we need to store this in Mirror? if we know a token is permissioned, cant we just ensure that it cant do ibc trasnfer to any chain except the source chain? i.e it can only return on the channel it came from, nothing else
  err = StoreAccountBlocklist(data) 
  if err != nil {
    return NewErrorAcknowledgement(err)
  }
  return NewAcknowledgement(result)
}

func StoreAccountBlocklist(data ics21types.SetAccountBlocklistPacket) {
  // Getting the current permissions on the Mirror module for the Permissioned Token denom
  denomPermissions = keeper.GetDenomPermissions(data.GetDenom())
  // Encode the Blocklist addition pubkeys 
  for var addedPubkey in data.GetAccountblocklistAdditions() {
    var addedAddress = EncodeToBech32(addedPubkey)
    denomPermissions.push(addedAddress)  
  } 
  // Encode the Blocklist removal pubkeys
  for var removedPubKey in data.GetAccountBlocklistRemovals() {
    var removedAddress = EncodeToBech32(addedPubkey)
    denomPermissions.pop(removedAddress)
  }
  keper.SetDenomPermissions(data.GetDenom(), denomPermissions)
}
```

`onAcknowledgePacket` need not be implemented for the Mirror chain as it does not issue any packets. (//todo should we consider sending back the actual modifications done to the permissions in the ack packet, in case only a couple of them failed encoding etc, so we keep the valid ones and ignore the invalid ones on the mirror side and let the Host know via ack packet on what actually happened) The host chain on `onAcknowledgePacket` checks if any error was found on the counterparty chain and resets its state to reflect the Mirror state.

```typescript
// Called on Host Chain by Relayer
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes
) {
  var data: ics21types.SetAccountBlocklistPacket
  data, err = packet.GetData() 
  if err != nil {
    return NewErrorAcknowledgement(ics21types.ErrInvalidType)
  }

  switch typeof(acknowledgement) {
    case *channeltypes.Acknowledgement_Error:
        RemoveAccountBlocklist(data) // todo: this is how the current implementation handles this, but in case of single Host but multiple Mirrors, this would mean, in case one channel failed to update but all others succeeded, the Host would still reset the state and end up with state mismatch.
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
  RemoveAccountBlocklist(data) // todo: same as above. in case of Ordered channels this would close the channel. should handle that. but in general, how to handle? re-attempt to send permissions update again?
}
```

### Host Chain Contract

#### **RegisterPermissionedToken**

`RegisterPermissionedToken` is the entrypoint to registering an existing token to be permissioned across IBC.
```typescript
function RegisterPermissionedToken (
  // denom is the existing token denom which will now be permissioned via ICS21
  denom: string,
  // owner is the address responsible for updating the permissions of the token
  owner: string
) {
  // Ensure the denom is registered with the x.bank module
  denomExists = GetDenomFromBank(denom)
  abortTransactionUnless(denomExists == true)
  
  // todo: check if the owner is a known tokenfactory owner. alt, expose this as a keeper to be called from within token factory and not as an explicit msg 
  SetPermissionedDenom(denom, owner)
}
```

#### **UpdateChannelAllowlist**

`UpdateChannelAllowlist` is the entrypoint to add and remove from the ChannelAllowlist for a particular denom and ICS20 channel IDs
```typescript
function UpdateChannelAllowlist (
  // sender is the owner of the token as set from the `RegisterPermissionedToken` 
  sender: string,
  // denom is the existing Permissioned token denom
  denom: string,
  // allowedChannels is a list of ICS20 channels IDs over which the denom can be sent over 
  allowedChannels: string[],
  // removedChannels is a list of ICS20 channel IDs over which the denom cannot be sent over anymore
  removedChannels: string[]
) {
  // Ensure only denom owner can update the allowlist
  owner = GetPermissionedDenomOwner(denom)
  abortTransactionUnless(owner == sender)

  pemissions = GetDenomPermission(denom)
  for var channel in allowedChannels {
    // Ensure the channel is of type ICS20
    channelInfo = GetIBCChannelInfo(channel)
    abortTransactionUnless(channelInfo.Port == "transfer")
    permissions.ChannelAllowlist.push(channel)
  }
  for var channel in removedChannels {
    permissions.ChannelAllowlist.pop(channel)
  }

  SetPermissions(denom, permissions)

  // todo: should we consider initiating a new ICS21 channel here itself for every allowedChannel? this would mean for every Permissioned Token, there will be a dedicated channel. We could store a mapping of denom -> ics21 channels across all chains. Easy lookup for when permissions need to be updated everywhere. This would also allow to use Ordered Channels without ending up in a situation where the channel closes for all due to one timeout. so it reduces surface area of that kinda issues. we could trigger channel closes for removedChannels too
}
```


#### **UpdateAccountBlocklist**

`UpdateAccountBlocklist` is the entrypoint used to add and remove pubkeys from the AccountsBlocklist for a particular denom
```typescript
function UpdateAccountBlocklist (
  // sender is the owner of the token as set from the `RegisterPermissionedToken` 
  sender: string,
  // denom is the existing Permissioned token denom
  denom: string,
  // addToBlocklist is a list of pubkeys which cannot interact with the denom 
  addToBlocklist: bytes[],
  // removeFromBlocklist is a list of pubkekys which can now interact with the denom
  removeFromBlocklist: bytes[]
) {
  // Ensure only denom owner can update the allowlist
  owner = GetPermissionedDenomOwner(denom)
  abortTransactionUnless(owner == sender)

  pemissions = GetDenomPermission(denom)
  for var pubkey in addToBlocklist {
    abortTransactionUnless(pubkey.Valid() == true)
    permissions.AccountBlocklist.push(pubkey)
  }
  for var pubkey in removeFromBlocklist {
    permissions.AccountBlocklist.pop(pubkey)
  }

  // Set the updated list locally
  SetPermissions(denom, permissions)

  channels = GetAllICS21Channels()
  updatePacket = ics21types.SetAccountBlocklistPacket {
    denom = denom,
    accountblocklist_additions = addToBlocklist
    accountblocklist_removals = removeFromBlocklist
  }
  for var channel in channels {
    handler.SendPacketOnChannel(channel, updatePacket)
  }
}
```

### Cosmos-SDK Contract

#### Antehandler

The ante handler functionality provided by the Cosmos-SDK allows the restriction of Permissioned Tokens from being sent across any channels except the channel it came from.
For every msg in a transaction, a check if performed to see if its a ICS20 transfer message. If it is, and the denom is a Permissioned Token denom, then the source port and source channel is checked against the channel the Permissioned Token came from. If they are the same, the transfer is allowed, else an error is returned.

```go
var _ sdk.AnteDecorator = ICS21Decorator{}

func (i ICS21Decorator) AnteHandle(ctx sdk.Context, tx sdk.Tx, simulate bool, next sdk.AnteHandler) (newCtx sdk.Context, err error) {
	msgs := tx.GetMsgs()
	for _, m := range msgs {
		switch msg := m.(type) {
		case *transfertypes.MsgTransfer:
			return ctx, i.HandleMsgTransfer(ctx, msg)
		}
	}
	return next(ctx, tx, simulate)
}

func (i ICS21Decorator) HandleMsgTransfer(ctx sdk.Context, msg *transfertypes.MsgTransfer) error {
  // Host: Ensure that permissioned denom is only sent on a whitelisted channel
  // Mirror: Ensure that an ICS21 denom can only be sent back to Host Chain and nowhere else
	denom = msg.Token.GetDenom()
  permissions = GetDenomPermissions(denom)
  // if given denom is not an ICS21 denom, continue with default behaviour
  if permissions == nil {
    return nil
  }
  if ics20types.SenderChainIsSource(msg.SourcePort, msg.SourceChannel, denom) {
    for channel range permissions.AllowedChannel {
      if msg.SourceChannel == channel {
        continue
      }
    }
    return errors.New("Attempting to send a permissioned token to a non whitelisted channel")
  }
  if ics20types.ReceiverChainIsSource(msg.SourcePort, msg.SourceChannel, denom) {
    return nil
  }
  return errors.New("Attempting to send an ICS21 token on a channel it did not come from")
}
```

#### SendRestrictionFn

The x/bank module in Cosmos-SDK v0.50.x onwards allows a protocol to provide custom SendRestrictions. This can be used to ensure that the AccountBlocklist is ensured for native transfers on the Mirror Chain.

`type SendRestrictionFn func(ctx context.Context, fromAddr, toAddr sdk.AccAddress, amt sdk.Coins) (newToAddr sdk.AccAddress, err error)`

```typescript
function SendRestrictionFn(
  ctx sdk.Context, 
  fromAddr sdk.AccAddress, 
  toAddr sdk.AccAddress, 
  amt sdk.Coins
) (newToAddr sdk.AccAddress, err error) {
  // Get the denom being transferred and its permissions
  denom = amt.GetDenom()
  permissions = GetDenomPermissions(denom)
  // Ensure the sender or the receiver are not part of the blocklist
  if permissions.AccountBlocklist.Contains(fromAddr) {
    return nil, throw new Error("Sender is in blocklist")
  }
  if permissions.AccountBlocklist.Contains(toAddr) {
    return nil, throw new Error("Receiver is in blocklist")
  }
  return toAddr, nil
}
```

### Properties & Invariants

(properties & invariants maintained by the protocols specified, if applicable)
- any ics21 denoms with multi hops

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

This initial standard uses version `"ics21-1"` in the channel handshake.

A future version of this standard could use a different version in the channel handshake, and safely alter the packet data format & packet handler semantics.

## Example Implementations

- An implementation of ICS 21 Host & Mirror in Golang can be found [here](https://github.com/noble-assets/ics21).
- An implementation of ICS 21 Mirror in [CosmWasm](https://cosmwasm.com) can be found [here](https://github.com/noble-assets/cw-ics21).

## Known Issues

ICS21 permissions will not apply to tokens which were sent before ICS21 was activated.

## Future Improvements

Handle trahsfer natively in the protocol insteaed of relying on ICS20 and building wrappers around it

## History

(changelog and notable inspirations / references)

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
