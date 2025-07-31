# SSRI UDT

**Author:** Hanssen  
**Date:** Jun 2024

## Introduction
Based on the SSRI, we can describe the behavior that Script of UDT types should have. If a Script declared as a UDT type does not have the following methods, it should be parsed according to the behavior described in the xUDT specification, and the unknown items should be completed by other methods.

## UDT Trait
### UDT.balance - Cell

```rust
fn balance() -> u128;
```

Retrieves the UDT balance stored in the Cell.

### UDT.name - Script

```rust
fn name() -> Bytes;
```

Retrieves the name of the UDT represented by the Script.

### UDT.symbol - Script

```rust
fn symbol() -> Bytes;
```

Retrieves the symbol of the UDT represented by the Script.

### UDT.decimals - Script

```rust
fn decimals() -> u8;
```

Retrieves the number of decimal places for the smallest unit of the UDT represented by the Script.
