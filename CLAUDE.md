# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a repository of CKB information which has been copied from various sources into a format easier for AI to read. The repository serves as a comprehensive knowledge base for CKB-related protocols, specifications, and technical documentation.

## Repository Structure

### Core Files
- `README.md` - Project description and comprehensive overview
- `LICENSE` - MIT License (Copyright 2025 Jordan Mack)
- `CLAUDE.md` - This guidance file for Claude Code

### CoBuild Protocol Documentation (`CoBuild/`)
Contains comprehensive documentation about the CKB Transaction CoBuild Protocol:

- `CKB Transaction CoBuild Protocol Overview.md` - Main protocol specification with goals, roles, basic flow, data schema, and examples
- `CKB Open Transaction (OTX) CoBuild Protocol Overview.md` - Open transaction protocol extension with off-chain construction and agent P2P networks
- `LockType Script Validation Rules.md` - Detailed validation rules for lock/type scripts supporting CoBuild
- `How to Improve Spore with CoBuild.md` - Practical implementation example using Spore NFT protocol
- `Custom Union Ids in WitnessLayout.md` - Technical specification for custom union identifiers
- `CoBuild-Only or CoBuild+Legacy.md` - Implementation guidelines for different CoBuild modes
- `CoBuild Hashes.md` - Cryptographic hash specifications and signing procedures

### SSRI Protocol Documentation (`SSRI/`)
Contains documentation about Script-Sourced Rich Information protocol:

- `Script-Sourced Rich Information.md` - Core SSRI specification enabling off-chain script execution with rich information return
- `SSRI UDT.md` - SSRI trait definitions for User Defined Token scripts (balance, name, symbol, decimals methods)

## Key Protocols Covered

### CoBuild Protocol
A collaborative transaction building framework enabling multiple parties to jointly construct CKB transactions with:
- Standardized roles (Builder, Asset Manager, Fee Manager, Signer)
- Message signing standards (similar to EIP712)
- Script interface standards for on-chain interactions
- Witness layout standards replacing WitnessArgs
- P2P-based, local-first architecture

### SSRI Protocol  
A script extension system allowing CKB scripts to:
- Execute off-chain and return structured results
- Provide rich information about script behavior
- Define method-based interfaces for external interaction
- Support trait-based behavioral abstractions

## Development Notes

This is primarily a documentation/reference repository with no build system, test suite, or development commands. The repository contains curated CKB technical information organized for optimal AI consumption and reference.

All markdown files have been formatted with proper headers, code blocks, and structure for enhanced readability while preserving technical accuracy.