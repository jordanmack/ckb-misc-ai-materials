# Appendix: CoBuild Hashes

**Author:** janx  
**Date:** Feb 2024

This section provides a detailed introduction to the various hashing algorithms used in the CoBuild protocol.

## Signing Message Hash

The term signing message hash is used here to represent the data being verified against the signature.

The implementation of Lock needs to consider the impact of wallet. Since CKB has the ability to implement other chains' transaction signing algorithms, CKB users may use wallets of other chains to operate assets on CKB. In such scenarios, a lock needs to consider how signing messages are calculated in non-CKB-specific wallet's signing processes. As we cannot modify these non-CKB-specific wallet's signing message calculation and they usually differ, there can't be a unified algorithm and process for calculating signing mesage hash in CKB lock script. Therefore, CoBuild protocol should only specify relevant data in CKB transactions that need coverage within signing message hash. A CoBuild-compatible lock script just needs ensure these necessary data are covered by signing message hash for security assurance.

When calculating signing message hash, a lock should first extract 'Message' based on whether it is under SighashAll or Otx mode. Under all circumstances, 'Message' must be covered by 'signing message hash'. The lock script also needs acquire some other transaction-related data such as current tx hash and certain cell content etc., then based on its mode it concatenates contents stipulated by CoBuild protocol, with information required by its own logic; after one or multiple cryptographic hashing it gets the 'signing message hash' for verification against the signature.

Although there isn't mandatory ordering requirement for contents like 'Message', tx hash and cell data that need coverage of signing message hash, different transactions should provide different preimage data for hashing function whenever possible.

### Avoiding Hash Collisions

For example: suppose a CoBuild-compatible lock's signing message hash covers 'Message' and certain cell data (note that in actual scenarios, data needing coverage by signing message hash is not limited to these, this is just a simple example), the lock chooses to concatenate 'Message' with cell data directly into a string then performs one hashing operation to obtain signing message hash. Then for following two situations, the generated signing message hash would be identical:

1. Message being 0xaabbccddaabbccdd, while cell data being 0x10101010.
2. Message being 0xaabbccdd, while cell data being 0xaabbccdd10101010.

This algorithm obviously isn't good enough as it allows possible transaction forgery since the same signing message preimages/hashes are constructed from different transactions and different Messages.

To avoid such problem, a common practice is for lock script to concatenate length of a piece of data at its front. For instance in above example, lengths of cell data are either 4 or 8. If we also include length information into hashing then these two cases will generate distinct signing messages hashes eventually.

Also note: Molecule-format data structures which possess canonical encoding property are widely used in CoBuild process. Simply put, a Molecule-format data structure either has fixed length or like Message where first four bytes already contains entire structure's length. Therefore there's no need manually prepend lengths onto such structures expressed in molecule format to avoid aforementioned issue.

## SighashAll Variant

In the SighashAll mode, the signing message hash should cover the following content:

1. The Message contained in the only SighashAll in the current transaction. Since Message is in molecule format, there' no need to include the length of the Message part in hashing.
2. The tx hash that can be obtained through CKB syscall. Note that tx hash is always 32 bytes and does not require additional length concatenation.
3. All input cells and data obtainable via CKB_syscall. As tx hash has covered inputs quantity, there's no need for further inputs count concatenation here. Input cell is presentedy as CellOutput in molecule format thus requires no additional length prefix. However, data is in variable-length raw format, to ensure safety it's necessary to prepend the length of data.
4. The lengths and contents of all unmatched witnesses that exceed the number of input cells.

When calculating signing message hash, a lock can concatenate all these contents together in a certain order, and further concatenate extra data needed by itself. Then the lock gets the signing message hash by feeding the concatenated string into a cryptographic hash function.

Note that the above hashing algorithm is just an example. In fact, CoBuild only requires lock script to ensure that certain data are covered by cryptographic hash. As long as it's defined clearly, any other hashing algorithm can be used and achieve the same security.

### Alternative Hashing Algorithm

For instance, for some (perhaps with special requirements) lock scripts, you could use the following algorithm:

1. First concatenate tx hash, all input cells and data along with lengths and content of all unmatched witnesses together; run a cryptographic hash function on this concatenated string and get a hash, let's call it skeleton hash.
2. Next, the lock script concatenates a fixed prefix with Message structure and skeleton hash, run another hash function and get another hash as signing message hash.

### SighashAll Example

Take the transaction below as an example:

![CoBuild Hashes 1](CoBuild%20Hashes%201.png)

A CoBuild-compatible lock script should cover the following content in its signing message hash during verification:

1. The Message contained in the transaction's SighashAll type witness
2. Current tx Hash
3. CellOutput corresponding to I1 serialized in molecule format
4. CellOutput Data corresponding to I1, with its length and actual content represented in little-endian encoding u32
5. CellOutput corresponding to I2 serialized in molecule format
6. CellOutput Data corresponding to I2, with its length and actual content represented in little-endian encoding u32
7. Length of Witness 3 and its actual content represented in little-endian encoding u32
8. Length of Witness 4 and its actual content represented in little-endian encoding u32
9. Length of Witness 5 and its actual content represented in little-endian encoding u32

## SighashAllOnly Variant

In addition to the SighashAll variant, there is a special SighashAllOnly variant of WitnessLayout. This variant only contains seal, not message. If all witnesses in a transaction are of the SighashAllOnly type, it means that this transaction does not contain any Message.

In this case, when a lock script verifies signatures, the following content should be covered in the signing message hash:

1. The tx hash obtained from CKB syscall.
2. All input cells and data obtained from CKB_syscall. Since tx hash has already covered inputs count, there's no need to include them again here.
3. The length and content of all unmatched witnesses exceeding input cell count.

If a lock script needs to support both modes with or without 'Message', different prefixes should be added or the built-in methods of hash algorithm (such as personalization provided by blake2b) should be used to distinguish transactions with 'Message' and without 'Message', so as to avoid generating the same signing message hash for these two different types of transactions.

## Otx Variant

In Otx mode, the signing message hash should cover:

1. The Message contained in the witness corresponding to current Otx.
2. The number of input cells covered by Otx, all content of input cells in CellInput format, including cell points in CellOutput format, and cell data in raw data form.
3. The number of output cells covered by Otx and all content of output cells including cell points in CellOutput format and cell data in raw data form.
4. The number of cell deps covered by Otx and all content of cell deps in CellDep format.
5. The number header deps covered by OTX and all contents represented using Byte32.

Note here cell data exists as variable-length raw strings. To ensure safety, a lock needs to prepend length before cell data. Besides this, other included structures are in molecule format which do not require additional length concatenation.

When calculating signing message hash, a lock can concatenate these contents together according to a certain order, and any extra data required by itself can also be added. Then the lock calculates get the result hash by feeding the concatenated string into a cryptographic hash function.

For security reasons it's recommended to distinguish signing preimages/hashes in SighashAll mode & Otx mode by using different prefixes or built-in methods from hashing functions (like Blake2B's personalization).

### Recommended Prefixes

More specifically here we suggest using different prefixes for the following three scenarios:

1. SighashAll mode transaction (with Message)
2. SighashAllOnly mode transaction (without Message)
3. Otx mode transaction

### Otx Example

Take the transaction below as example:

![CoBuild Hashes 2](CoBuild%20Hashes%202.jpg)

When a CoBuild-compatible lock script is verifying the transaction, the following content should be covered in calculated signing message hash:

1. The unique Message contained in the Otx typed witness of the open transaction.
2. 2 (the number of input cells as little endian encoded u32)
3. I1's CellInput
4. I1's CellOutput Data in molecule format, including data length and actual content as u32 in little-endian encoding
5. I2's CellInput
6. I2's CellOutput Data in molecule format, including data length and actual content as u32 in little-endian encoding
7. 3 (the number of output cells, as u32 in little endian encoding)
8. O1's CellOutput
9. Length of O1 cell data
10. Actual content of O1 cell data
11. O2's CellOutput
12. Length of O2 cell data
13. Actual content of O2 cell data
14. O3's CellOutput
15. Length of O3 cell data
16. Actual content of O3 cell data
17. '1' (the number of cell deps as u32 in little endian encoding)
18. D1's CellDep
19. 4 (the number of header deps as u32 in little endian encoding)
20. H1 as Byte32
21. H2 as Byte32
22. H3 as Byte32
23. H4 as Byte32

## Example Implementation

This repository provides an example implementation of signing message hash.

This example shows how to concatenate the data required by the CoBuild specification and obtain the result signing message hash through a blake2b hash calculation in three different modes.

For three different modes in the example, we have chosen different blake2b personalizations to avoid mistakenly generating the same hash for different types of transactions:

- **For SighashAll mode, with Message in transaction:** use `ckb-tcob-sighash`
- **For SighashAllOnly mode, without Message in transaction:** use `ckb-tcob-sgohash` ('sgo' means 'sighash only')
- **For Otx mode:** use `ckb-tcob-otxhash`

As mentioned above, this is just one example implementation of a signing message hash. For a lock script to be CoBuild compatible and hashing securely, it should make sure that:

1. All data required by the CoBuild protocol are correctly covered,
2. There is no ambiguity in data concatenation, and
3. The signing hash is generated using cryptographic hashing functions.