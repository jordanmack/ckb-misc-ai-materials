# CKB Miscellaneous AI Materials

This is a repository of CKB information which has been copied from various sources into a format easier for AI to read. The repository serves as a comprehensive knowledge base for CKB-related protocols, specifications, and technical documentation.

## Repository Structure

### CoBuild Protocol Documentation
The `CoBuild/` directory contains comprehensive documentation about the CKB Transaction CoBuild Protocol:

- **CKB Transaction CoBuild Protocol Overview.md** - Main protocol specification describing collaborative transaction building procedures, roles, data structures, and workflows
- **CKB Open Transaction (OTX) CoBuild Protocol Overview.md** - Extension protocol for open transactions, including off-chain construction and agent P2P networks  
- **LockType Script Validation Rules.md** - Detailed validation rules for lock and type scripts supporting CoBuild protocol
- **How to Improve Spore with CoBuild.md** - Practical example of implementing CoBuild for Spore NFT protocol
- **Custom Union Ids in WitnessLayout.md** - Technical specification for custom union identifiers in witness layouts
- **CoBuild-Only or CoBuild+Legacy.md** - Guidelines for implementing CoBuild-only vs hybrid CoBuild+legacy modes
- **CoBuild Hashes.md** - Cryptographic hash specifications for CoBuild protocol implementations

### SSRI (Script-Sourced Rich Information) Documentation  
The `SSRI/` directory contains documentation about extending script capabilities:

- **Script-Sourced Rich Information.md** - Core SSRI specification enabling scripts to execute off-chain and return rich information
- **SSRI UDT.md** - SSRI trait definitions for User Defined Token (UDT) scripts, including balance, name, symbol, and decimals methods

## Content Overview

This repository focuses on two major CKB protocol innovations:

1. **CoBuild Protocol** - A collaborative transaction building framework that enables multiple parties to jointly construct CKB transactions with standardized data exchange formats and verification procedures.

2. **SSRI Protocol** - A script extension system that allows CKB scripts to provide rich information and execute off-chain while maintaining on-chain verification capabilities.

All documentation has been formatted for optimal AI readability while preserving technical accuracy and completeness.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
