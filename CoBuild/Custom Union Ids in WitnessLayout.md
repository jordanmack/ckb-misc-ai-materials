# Appendix: Custom Union Ids in WitnessLayout

**Author:** janx  
**Date:** Feb 2024

In the WitnessLayout structure, each union variant has a special custom union id (a larger magic number). This is a trick used to maintain forward compatibility: such special values allow WitnessLayout and previous WitnessArgs to coexist without affecting each other.

## WitnessArgs vs WitnessLayout Serialization

The widely used WitnessArgs is a molecule table containing three fields. Its serialized data looks roughly like this:

```
<length> <offset 1> <offset 2> <offset 3> ...
```

Both length and offset are of little endian u32 type.

And the serialization of a molecule union looks something like this:

```
<union id> ....
```

Here, the union id is also of little endian u32 type.

## Custom Union ID Range

For WitnessLayout, it currently uses custom union ids such as 0xFF000001, 0xFF000002. Considering future expansion, WitnessLayout should only use the range from 0xFF000001 to 0xFFFFFFFF for new variants' union ids (this range has over 16 million values which are enough).

## Why This Works

**Why?** Assuming there's some serialized data in the format of WitnessLayout, if you read out its first four bytes as little endian u32, the result will definitely not be less than 0xFF000001. If you try to parse this data using the WitnessArgs type, these first four bytes would be read out as length for whole data. For example, 0xFF000001 == 4278190081 ~=4080MB. But CKB block size maxes at around 600k, even considering future expansion, it's hard to reach 4000MB. Therefore, it is practically impossible to construct a WitnessArgs of length 0xFF000001 or longer. So if you parse the binary data serialized in the format of WitnessLayout as WitnessArgs, the molecule verification will definitely fail.

Similarly, for data serialized in the format of WitnessArgs, the value read from its first four bytes would be far less than 0xFF000001, and could not possibly be any union id used by WitnessLayout. Thus if you try parsing this data as WitnessLayout, molecule verification will also certainly fail.

## Practical Benefits

Therefore, in transaction witnesses you can arbitrarily insert either WitnessArgs formatted or WitnessLayout formatted data. On-chain scripts can deterministically determine what format a certain witness uses. Data in WitnessLayout format will surely fail validation when parsed as WitnessArgs, and vice versa. In future we can further expand WitnessLayout to support more variants.

## Summary

If we look at the encoding, the WitnessLayout union places a 4-byte ID in front of the actual witness content. This 4-byte ID works like a version flag, helps differentiate new format from existing WitnessArgs.