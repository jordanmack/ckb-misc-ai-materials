# Example: How to Improve Spore with CoBuild

**Author:** janx  
**Date:** Feb 2024

Here we use Spore as an example to demonstrate how to add corresponding Message and other CoBuild data structures for Spore according to the CoBuild specifications.

## Spore Message

For Spore, we can define the following Message schema:

```c
// Common types are omitted here for simplicity

table Mint {
    id: Byte32,
    to: Address,
    content_hash: Byte32,
}

option AddressOpt (Address);

table Transfer {
    nft_id: Byte32,
    from: AddressOpt,
    to: AddressOpt,
}

table Melt {
    id: Byte32,
}

union SporeAction {
    Mint,
    Transfer,
    Melt,
}
```

and ScriptInfo like this:

| Field | Value |
|-------|-------|
| name | Spore |
| url | |
| script_hash | |
| schema | |
| message_type | SporeAction |

The above ScriptInfo structure will be serialized by molecule, and hashed by ckb-hash, let's call the obtained result `<spore script info hash>`.

A spore cell using JoyID lock has the following data structure:

- **Lock:** JoyID lock
- **Type:** Spore Type Script
- **Data:** Spore Data

The structure of a Spore cell remains completely unchanged as when not using CoBuild Message. What changed is the witness in transactions involving Spore cells.

## Mint

How Mint operation is implemented depends on the capability of input cells' locks. There are two methods to build a Spore mint transaction: using legacy structure based on WitnessArgs, and using WitnessLayout structure and CoBuild flow.

### Legacy Format Example

Assuming that lock only supports the legacy format WitnessArgs, a mint transaction can be constructed as follows:

```
inputs:
  input 0:
    capacity: 1000 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
  input 1:
    capacity: 500 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
  input 2:
    capacity: 300 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
outputs:
  output 0:
    capacity: 1600 CKB
    lock: JoyID lock B
    type: Spore type script
    data: Spore data
  output 1:
    capacity: 199 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
witnesses:
  witness 0: WitnessArgs format
    lock: Signature for JoyID lock A
    input_type: <EMPTY>
    output_type: <EMPTY>
  witness 1: <EMPTY>
  witness 2: <EMPTY>
  witness 3: WitnessLayout format, SighashAll variant
    seal: []
    message: Message format
      actions:
        Action 0:
          script_info_hash: <spore script info hash>
          script_hash: <spore type script hash>
          data: SporeAction format, Mint variant
            id: <Spore id of output 0>
            to: JoyID lock
            content_hash: <hash of Spore data of output 0>
```

### CoBuild Format Example

If the lock used by input cells supports CoBuild, a transaction with the following structure can be used:

```
inputs:
  input 0:
    capacity: 1000 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
  input 1:
    capacity: 500 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
  input 2:
    capacity: 300 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
outputs:
  output 0:
    capacity: 1600 CKB
    lock: JoyID lock B
    type: Spore type script
    data: Spore data
  output 1:
    capacity: 199 CKB
    lock: JoyID lock A
    type: <EMPTY>
    data: <EMPTY>
witnesses:
  witness 0: WitnessLayout format, SighashAll variant
    seal: Signature for JoyID lock A
    message: Message format
      actions:
        Action 0:
          script_info_hash: <spore script info hash>
          script_hash: <spore type script hash>
          data: SporeAction format, Mint variant
            id: <Spore id of output 0>
            to: JoyID lock
            content_hash: <hash of Spore data of output 0>
```

### BuildingPacket Example

Assume that the current transaction is `<spore tx 1>`. Spore dapp can build and send the following packet to JoyID:

```
BuildingPacket format, BuildingPacketV1 variant
  message: Message format
    actions:
      Action 0:
        script_info_hash: <spore script info hash>
        script_hash: <spore type script hash>
        data: SporeAction format, Mint variant
          id: <Spore NFT id of output 0>
          to: JoyID lock
          content_hash: <hash of Spore NFT data of output 0>
  payload: <spore tx 1 without witness 0>
  script_infos: Array
    0: <Spore ScriptInfo defined above>
  lock_actions: <EMPTY>
```

Here we first present the mint transaction structure, then give the BuildingPacket structure. But in actual flow, it's the Spore app generating BuildingPacket first, then sending it to JoyID wallet which presents it to user. After user confirms and signs, JoyID generates signature and sends it back to the Spore app which fills received siganture in, complete and finalize the minting transaction. There's a BuildingPacket before the finalized minting transaction.

### Wallet Display Information

The following data in BuildingPacket should be shown to end users by wallet:

1. The actions in message. In this Spore example since there's only one spore type script in transaction so actions has only one member. For future more complex transactions (like buying spores with UDT), actions may contain many members.
2. The transaction fee/feerate
3. CKByte transfer info

### Notes on Implementation

The above two transaction examples implement the same Mint function. Some notes:

- Lock scripts in input cells use signatures in lock fields (for different type scripts, get from different locations in witnesses) to verify the transaction. In the two examples the methods of calculating signing message are slightly different:
  - In the transaction using WitnessArgs, the classical sighash-all mode is used to calculate signing message;
  - In the transaction using WitnessLayout, it follows CoBuild specification to calculate signing message.
- Someone can mint a Spore for himself. The A minting for B example demonstrates the usage of 'to' field within 'SporeAction'.
- Spore type script should extract 'message' from WitnessLayout and verify it matches the transaction data, e.g. verify if the state transition presented by the transaction is a Spore mint.
- For minting there might be cases in which transactions with the old WitnessArgs are used, and we use minting as an example to show how to be backward compatible with two different transaction structure. For other Spore actions we suppose both lock and type script used support new version transaction format based on WitnessLayout.

## Transfer

Transfer transaction should be like:

```
inputs:
  input 0:
    capacity: 1600 CKB
    lock: JoyID lock B
    type: Spore type script
    data: Spore 111
outputs:
  output 0:
    capacity: 1599.9 CKB
    lock: JoyID lock C
    type: Spore type script
    data: Spore 111
witnesses:
  witness 0: WitnessLayout format, SighashAll variant
    seal: Signature for JoyID lock B
    message: Message format
      actions:
        Action 0:
          script_info_hash: <spore script info hash>
          script_hash: <spore type script hash>
          data: SporeAction format, Transfer variant
            id: <ID of Spore 111>
            from: JoyID lock B
            to: JoyID lock C
```

Assuming the current transaction is `<spore tx 2>`, a Spore dapp should construct the following data, and send it to JoyID for display and request signature:

```
BuildingPacket format, BuildingPacketV1 variant
  message: Message format
    actions:
      Action 0:
        script_info_hash: <spore script info hash>
        script_hash: <spore type script hash>
        data: SporeAction format, Transfer variant
          id: <ID of Spore NFT 111>
          from: JoyID lock B
          to: JoyID lock C
  payload: <spore tx 2 without witness 0>
  script_infos: Array
    0: <Spore ScriptInfo defined above>
  lock_actions: <EMPTY>
```

Similar to Mint, here JoyID should present three sets of information to users:

1. The information in message (especially in message.data) which reprensets the actual operations will be carried out by this transaction.
2. Transaction fee/feerate (calculated from data in payload).
3. CKBytes transfer (also derived from data in payload).

After user confirmation, JoyID can generate signing message hash according to CoBuild spec and sign.

### Multi-Lock Transfer Example

If a transaction's input cells use multiple (e.g. two) locks, more signatures are required:

```
inputs:
  input 0:
    capacity: 1600 CKB
    lock: JoyID lock B
    type: Spore NFT type script
    data: Spore NFT 111
  input 1:
    capacity: 400 CKB
    lock: JoyID lock B
    type: <EMPTY>
    data: <EMPTY>
  input 2:
    capacity: 300 CKB
    lock: JoyID lock D
    type: <EMPTY>
    data: <EMPTY>
outputs:
  output 0:
    capacity: 1650 CKB
    lock: JoyID lock C
    type: Spore NFT type script
    data: Spore NFT 111
  output 1:
    capacity: 349.9 CKB
    lock: JoyID lock B
    type: <EMPTY>
    data: <EMPTY>
  output 2:
    capacity: 300 CKB
    lock: JoyID lock D
    type: <EMPTY>
    data: <EMPTY>
witnesses:
  witness 0: WitnessLayout format, SighashAll variant
    seal: Signature for JoyID lock B
    message: Message format
      actions:
        Action 0:
          script_info_hash: <spore script info hash>
          script_hash: <spore type script hash>
          data: SporeAction format, Transfer variant
            id: <ID of Spore NFT 111>
            from: JoyID lock B
            to: JoyID lock C
  witness 1: <EMPTY>
  witness 2: WitnessLayout format, SighashAllOnly variant
    seal: Signature for JoyID lock D
```

In a CoBuild transaction, only one witness can use SighashAll format containing Message. For transactions that require signatures from multiple different locks, only one lock's corresponding witness can use SighashAll format with Message included; other lock's corresponding witnesses should use SighashAllOnly format that contains only signature.

There will be only one 'Message' in a CoBuild transaction. Even if there are multiple locks / type scripts, all actions required by these locks / type scripts should be placed within this same 'Message'.

Taking above transaction as an example: input cell #0 and #1 uses JoyID lock B while input cell #2 uses JoyID lock D. According to CKB verification rules we need provide signatures for both JoyID lock B and D within witnesses. Here we choose put 'Message' into witness 0 which corresponds with JoyID Lock B, whereas witness #2 which corresponds with JoyID Lock D used SighashAllOnly format.

Actually when signing using JoyID Locks B & D they're using exactly same signing message hash. In other words when JoyID Lock D signs it still needs to cover 'Message' stored in witness corresponding with JoyID Lock B.

Assuming the current transaction is `<spore tx 3>`, a Spore dapp should generate same BuildingPacket and send separately to JoyID lock B and D for signature:

```
BuildingPacket format, BuildingPacketV1 variant
  message: Message format
    actions:
      Action 0:
        script_info_hash: <spore script info hash>
        script_hash: <spore type script hash>
        data: SporeAction format, Transfer variant
          id: <ID of Spore 111>
          from: JoyID lock B
          to: JoyID lock C
  payload: <spore tx 3 without witness 0>
  script_infos: Array
    0: <Spore ScriptInfo defined above>
  lock_actions: <EMPTY>
```

In addition, because input cells with JoyID lock B use witnesses at index #0, #1, witness at index #1 must be empty. Witness corresponding with JoyID lock D should be placed at index #2.

## Melt

Melt transaction:

```
inputs:
  input 0:
    capacity: 1600 CKB
    lock: JoyID lock B
    type: Spore type script
    data: Spore 111
outputs:
  output 0:
    capacity: 1599.9 CKB
    lock: JoyID lock C
    type: <EMPTY>
    data: <EMPTY>
witnesses:
  witness 0: WitnessLayout format, SighashAll variant
    lock: Signature for JoyID lock B
    message: Message format
      actions:
        Action 0:
          script_info_hash: <spore script info hash>
          script_hash: <spore type script hash>
          data: SporeAction format, Melt variant
            id: <ID of Spore 111>
```

Assuming the current transaction is `<spore tx 4>`, a Spore dapp should construct the following data, send it to JoyID wallet for display and request a signature:

```
BuildingPacket format, BuildingPacketV1 variant
  message: Message format
    actions:
      Action 0:
        script_info_hash: <spore script info hash>
        script_hash: <spore type script hash>
        data: SporeAction format, Melt variant
          id: <ID of Spore NFT 111>
  payload: <spore tx 4 without witness 0>
  script_infos: Array
    0: <Spore ScriptInfo defined above>
  lock_actions: <EMPTY>
```