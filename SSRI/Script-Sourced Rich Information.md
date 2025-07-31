# Script-Sourced Rich Information

**Author:** Hanssen  
**Date:** Jun 2024

## Background
Currently, scripts on CKB are usually used as verifiers. Scripts only allow transactions that meet the rules to pass, thereby controlling the state transitions of cells. Under this paradigm, the script itself exposes almost no logic externally, and more information about the script can only be defined upon off-chain or through loosely bound on-chain methods.

As more applications emerge on CKB, issues gradually surface. These include how UDT assets declare their token information, how NFT assets define their data structures, and how to solve the problem of implementing transaction assembly logic in different languages repeatedly. We urgently need strongly bound information/logic in scripts to address interoperability issues between applications.

At this point, the DOB protocol’s off-chain execution of the decoder mode reveals a simple and effective path for us.

## Introduction
The Script-Sourced Rich Information (SSRI) solution is an extension of the script’s capabilities. As its most significant feature, the script can execute off-chain and return results, allowing developers to embed arbitrary information/logic into the script, describing its behavior and managing data to off-chain applications.

## Design

The SSRI solution consists of two parts:

- **On-chain:** SSRI defines a behavior standard for scripts. Similar to traits in Rust, SSRI allows us to abstract behaviors into a series of method constraints. Scripts that meet these constraints can be considered to have specific behaviors. Callers can interact with the script through specified methods.
- **Off-chain:** SSRI defines a standard for script execution. Off-chain scripts can obtain more information sources through specific syscalls to construct outputs.
## Methods

### Method Path
In the general behavior standard of SSRI, callers invoke a specific method of the script through a 64-bit (8-byte) pointer called a method path.

We define that the method path is the first 8 bytes of the method signature's CKB hash. The format of the method signature is `[<trait>.]<method>`, such as `calc_unlock_time` or `UDT.symbol`. We strongly recommend that common abstract behavior methods should have a trait prefix to avoid conflicts. Since method type parameter does not affect method paths, the Script method does not support overloading, just like in Rust.

### Distribution

To achieve controlling script behavior through method paths, we notice that the CKB VM program entry is defined as follows:

```rust
fn __ckb_std_main(argc: core::ffi::c_int, argv: *const $crate::env::Arg) -> i8;
```
Like the C program entry, `argc` and `argv` together describe the program parameters.

> **Note:** The Rust package for CKB scripts `ckb-std` provides the `ckb_std::env::argv` method to access these parameters.

We define that scripts complying with SSRI should read the method path from `argv[0]` and pass the remaining parameters to the method. Developers can decide how to handle undefined situations, including unspecified method paths, incorrect method path lengths, and non-existent methods. We recommend:

- If no method path is specified, the script can execute the original verifier logic.
- If the method path length is incorrect or the method does not exist, the script should exit with any error code.
## Off-Chain Syscalls

Off-chain scripts still execute in the CKB VM. Below are currently available syscalls for off-chain execution and their differences from the on-chain version. Detailed introductions can be found in VM Syscalls, VM Syscalls 2, and VM Syscalls 3.

Off-chain scripts do not need to be attached to any entity, meaning scripts do not always call all syscalls. We categorize the execution environment into four levels:

- **Code:** Only the program code.
- **Script:** The complete script structure.
- **Cell:** The complete cell structure.
- **Transaction:** The complete transaction structure.
### VM Version - Code

Currently, if a script is executed off-chain, the VM version syscall should return -1.

### Exit - Code

### Set Content - Code

The maximum output length is determined by the execution environment and is no less than 256K.

### Debug - Code

### Load Script Hash - Script

### Load Script - Script

### Load Cell - Cell

If the execution level is Cell, the index must be 0, and the source must be `Source::InputGroup`.

### Load Cell By Field - Cell

If the execution level is Cell, the index must be 0, and the source must be `Source::InputGroup`.

### Load Cell Data - Cell

If the execution level is Cell, the index must be 0, and the source must be `Source::InputGroup`.

### Find Out Point By Type - Code

```c
int ckb_find_out_point_by_type(void* addr, size_t* len,
                               const void* type_script, size_t script_len)
{
  return syscall(u64_le_from_bytes(method_path("find_out_point_by_type")), addr, len, type_script, script_len, 0, 0);
}
```

This syscall is specifically designed for off-chain script execution. It searches for a live cell with the specified type script on-chain. If found, it stores the out point of the first found cell (note that the result may not always be consistent) in `addr`, with `len` being 36. Otherwise, `len` is 0.

### Find Cell By Out Point - Code

```c
int ckb_find_cell_by_out_point(void* addr, size_t* len, const void* out_point)
{
  return syscall(u64_le_from_bytes(method_path("find_cell_by_out_point")), addr, len, out_point, 0, 0, 0);
}
```

This syscall is specifically designed for off-chain script execution. It searches for a live cell on-chain by out point. If found, it stores the cell output in `addr`. Otherwise, `len` is 0.

### Find Cell Data By Out Point - Code

```c
int ckb_find_cell_data_by_out_point(void* addr, size_t* len, const void* out_point)
{
  return syscall(u64_le_from_bytes(method_path("find_cell_data_by_out_point")), addr, len, out_point, 0, 0, 0);
}
```

This syscall is specifically designed for off-chain script execution. It searches for a live cell on-chain by out point. If found, it stores the cell data in `addr`. Otherwise, `len` is 0.

## SSRI Trait
All scripts using the SSRI solution should implement these methods to facilitate their use. In the SSRI trait design, we default to using Rust’s type descriptions for parameters and return values:

- Numeric types are encoded in little-endian.
- Boolean types occupy a single byte, with 0 being false and any other value being true.
- Strings are encoded in utf-8 and stored as `vector<byte>` in molecule.
- Composite types use molecule encoding.
### SSRI.version - Code

```rust
fn version() -> u8;
```

Currently, this method should always return 0.

### SSRI.get_methods - Code

```rust
fn get_methods(offset: u64, limit: u64) -> Vec<Bytes8>;
```

Used to enumerate all methods of the script. The input parameters `offset` and `limit` are for pagination. If `limit` is 0, there is no restriction. It returns the corresponding method path array.

### SSRI.has_methods - Code

```rust
fn has_methods(methods: Vec<Bytes8>) -> Vec<bool>;
```

Determines whether a set of methods exists. The parameter is the target method path array.

### SSRI.get_cell_deps - Code

```rust
fn get_cell_deps(offset: u64, limit: u64) -> Vec<CellDep>;
```

Used to enumerate the cell deps required by the script during on-chain execution (excluding itself). The input parameters `offset` and `limit` are for pagination. If `limit` is 0, there is no restriction. It returns the corresponding cell dependencies array.
