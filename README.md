# Cardano Accumulator SDK

A Rust/JavaScript/Aiken SDK for creating, managing, and verifying cryptographic accumulators (Merkle trees, MMR, Sparse Merkle, Patricia Forestry) on the Cardano blockchain.

## Requirements

- **Create merklized structures** in Rust/Javascript
- **Support multiple merkle algorithms** for flexibility in implementation
- **Support membership and non-membership proofs** (optional feature)
- **Create universal transcription to JSON** for interoperability
- **CLI tool to verify onchain** with transcription

## Features

- Merklized data structure creation and management
- Multiple merkle algorithm support
- JSON serialization for universal compatibility
- Command-line interface for onchain verification
- Optional membership/non-membership proof support

## Project Structure

```
cardano-accumulator-sdk/
├── Cargo.toml          # Rust project configuration
├── src/
│   └── lib.rs          # Core library implementation
├── doc/
│   └── design.excalidraw  # Design document
└── README.md           # This file
```

## Getting Started

### Prerequisites

- Rust (see `Cargo.toml` for MSRV)

### Installation

```bash
cargo build
```

### Usage

```bash
# Build the project
cargo build --release

# Run tests
cargo test
```

## License

[Add license information here]

## Contributing

[Add contribution guidelines here]
