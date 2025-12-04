# Solana Wallet Adapter for Rust

<div align="center">

![Rust](https://img.shields.io/badge/rust-%23000000.svg?style=for-the-badge&logo=rust&logoColor=white)
![WebAssembly](https://img.shields.io/badge/WebAssembly-654FF0?style=for-the-badge&logo=WebAssembly&logoColor=white)
![Solana](https://img.shields.io/badge/Solana-9945FF?style=for-the-badge&logo=solana&logoColor=white)

[![crates.io](https://img.shields.io/crates/v/wallet-adapter.svg)](https://crates.io/crates/wallet-adapter)
[![Docs](https://docs.rs/wallet-adapter/badge.svg)](https://docs.rs/wallet-adapter)
[![License](https://img.shields.io/crates/l/wallet-adapter)](LICENSE)
[![Rust](https://github.com/JamiiDao/SolanaWalletAdapter/actions/workflows/rust.yml/badge.svg?branch=master)](https://github.com/JamiiDao/SolanaWalletAdapter/actions/workflows/rust.yml)

**A lightweight, type-safe Solana wallet adapter for Rust-based frontends and WebAssembly applications**

[Documentation](#documentation) • [Features](#features) • [Quick Start](#quick-start) • [Examples](#examples) • [Contributing](#contributing)

</div>

---

## 📖 Overview

`wallet-adapter` is a pure Rust implementation of the [Solana Wallet Standard](https://github.com/wallet-standard/wallet-standard), designed for building WebAssembly-based dApps with frameworks like Dioxus, Yew, and Sycamore. It provides a type-safe, async-first API for connecting to browser wallet extensions and interacting with the Solana blockchain.

### Key Features

- ✅ **Full Wallet Standard Compliance** - Implements all standard wallet features
- ✅ **Type-Safe API** - Leverages Rust's type system for compile-time safety
- ✅ **Async/Await Support** - Built on async primitives for modern Rust applications
- ✅ **Zero Unsafe Code** - `#![forbid(unsafe_code)]` enforced throughout
- ✅ **Framework Templates** - Ready-to-use templates for Dioxus, Yew, and Sycamore
- ✅ **Comprehensive Error Handling** - Detailed error types for all failure modes
- ✅ **Event-Driven Architecture** - Reactive wallet state management

## 🚀 Quick Start

### Installation

Add `wallet-adapter` to your `Cargo.toml`:

```toml
[dependencies]
wallet-adapter = "1.1.2"
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "Event",
    "EventTarget",
    "CustomEvent",
] }
wasm-bindgen-futures = "0.4"
```

### Basic Usage

```rust
use wallet_adapter::{WalletAdapter, WalletResult};

async fn connect_wallet() -> WalletResult<()> {
    // Initialize the wallet adapter
    let mut adapter = WalletAdapter::init()?;
    
    // Connect to a wallet by name
    adapter.connect_by_name("Phantom").await?;
    
    // Get connection info
    let connection = adapter.connection_info().await;
    let account = connection.connected_account()?;
    
    println!("Connected to: {}", account.address());
    
    Ok(())
}
```

## 📚 Documentation

### Comprehensive Guides

- **[Full Documentation Book](https://jamiidao.github.io/SolanaWalletAdapter/)** - Complete guide with examples
- **[API Documentation](https://docs.rs/wallet-adapter)** - Auto-generated API docs
- **[Templates Guide](templates/README.md)** - Framework-specific templates

### Core Concepts

#### Initialization

The adapter can be initialized in several ways:

```rust
// Default initialization (channel capacity: 5)
let adapter = WalletAdapter::init()?;

// Custom channel capacity
let adapter = WalletAdapter::init_with_channel_capacity(10)?;

// With custom Window and Document
let window = web_sys::window().unwrap();
let document = window.document().unwrap();
let adapter = WalletAdapter::init_custom(window, document)?;
```

#### Wallet Events

Listen for wallet events asynchronously:

```rust
let adapter = WalletAdapter::init()?;
let event_receiver = adapter.events();

while let Ok(event) = event_receiver.recv().await {
    match event {
        WalletEvent::Registered(wallet) => {
            println!("Wallet registered: {}", wallet.name());
        }
        WalletEvent::Connected(account) => {
            println!("Connected: {}", account.address());
        }
        WalletEvent::Disconnected => {
            println!("Wallet disconnected");
        }
        _ => {}
    }
}
```

#### Connecting to Wallets

```rust
// Connect by wallet name
adapter.connect_by_name("Phantom").await?;

// Or iterate through available wallets
for wallet in adapter.wallets() {
    if wallet.name() == "Phantom" {
        adapter.connect(wallet).await?;
        break;
    }
}
```

#### Checking Wallet Features

```rust
// Check cluster support
let supports_mainnet = adapter.mainnet().await?;
let supports_devnet = adapter.devnet().await?;

// Check feature support
let supports_sign_message = adapter.solana_sign_message().await?;
let supports_sign_transaction = adapter.solana_sign_transaction().await?;
let supports_sign_and_send = adapter.solana_sign_and_send_transaction().await?;
```

## 💡 Examples

### Sign In With Solana (SIWS)

```rust
use wallet_adapter::{WalletAdapter, SigninInput, Cluster};

async fn sign_in() -> WalletResult<()> {
    let mut adapter = WalletAdapter::init()?;
    adapter.connect_by_name("Phantom").await?;
    
    let public_key = adapter.connection_info().await
        .connected_account()?.public_key();
    let address = adapter.connection_info().await
        .connected_account()?.address().to_string();
    
    let mut signin_input = SigninInput::new();
    signin_input
        .set_domain(&adapter.window())?
        .set_statement("Login to My DApp")
        .set_chain_id(Cluster::DevNet)
        .set_address(&address)?;
    
    let signin_output = adapter.sign_in(&signin_input, public_key).await?;
    
    Ok(())
}
```

### Sign Message

```rust
async fn sign_message() -> WalletResult<()> {
    let mut adapter = WalletAdapter::init()?;
    adapter.connect_by_name("Phantom").await?;
    
    if adapter.solana_sign_message().await? {
        let output = adapter.sign_message(b"Hello, Solana!").await?;
        println!("Signature: {}", output.signature);
    }
    
    Ok(())
}
```

### Sign Transaction

```rust
use solana_sdk::{
    native_token::LAMPORTS_PER_SOL,
    pubkey::Pubkey,
    system_instruction,
    transaction::Transaction,
};
use wallet_adapter::{WalletAdapter, Cluster};

async fn sign_transaction() -> WalletResult<()> {
    let mut adapter = WalletAdapter::init()?;
    adapter.connect_by_name("Phantom").await?;
    
    let public_key = adapter.connection_info().await
        .connected_account()?.public_key();
    let pubkey = Pubkey::new_from_array(public_key);
    
    let recipient = Pubkey::new_unique();
    let instruction = system_instruction::transfer(&pubkey, &recipient, LAMPORTS_PER_SOL);
    let tx = Transaction::new_with_payer(&[instruction], Some(&pubkey));
    let tx_bytes = bincode::serialize(&tx)?;
    
    let signed_tx = adapter.sign_transaction(&tx_bytes, Some(Cluster::DevNet)).await?;
    
    Ok(())
}
```

### Sign and Send Transaction

```rust
use wallet_adapter::{WalletAdapter, Cluster, SendOptions};

async fn sign_and_send() -> WalletResult<()> {
    let mut adapter = WalletAdapter::init()?;
    adapter.connect_by_name("Phantom").await?;
    
    // ... build transaction ...
    
    let send_options = SendOptions::default();
    let signature = adapter.sign_and_send_transaction(
        &tx_bytes,
        Cluster::DevNet,
        send_options
    ).await?;
    
    println!("Transaction signature: {}", signature);
    
    Ok(())
}
```

## 🎨 Framework Templates

Ready-to-use templates for popular Rust frontend frameworks:

- **[Dioxus Template](templates/dioxus-adapter/)** - Full-featured Dioxus integration
- **[Yew Template](templates/yew-adapter/)** - Yew component library
- **[Sycamore Template](templates/sycamore-adapter/)** - Sycamore reactive components
- **[Anchor Integration Templates](templates/)** - Templates with Anchor program integration

Each template includes:
- Complete wallet connection UI
- Transaction signing flows
- Account management
- Network switching
- Error handling
- Modern, responsive design

## 🔧 Features

### Wallet Standard Features

- ✅ `wallet-standard:register-wallet` - Wallet registration
- ✅ `wallet-standard:app-ready` - App readiness notification
- ✅ `standard:connect` - Connect to wallet
- ✅ `standard:disconnect` - Disconnect from wallet
- ✅ `standard:events` - Wallet event subscription

### Solana-Specific Features

- ✅ `solana:signIn` - Sign In With Solana (SIWS)
- ✅ `solana:signMessage` - Message signing
- ✅ `solana:signTransaction` - Transaction signing
- ✅ `solana:signAndSendTransaction` - Sign and send transactions

### Additional Features

- ✅ Multi-cluster support (Mainnet, Devnet, Testnet, Localnet)
- ✅ Commitment level configuration
- ✅ Wallet account parsing and validation
- ✅ Ed25519 signature verification
- ✅ In-memory wallet storage
- ✅ Comprehensive error types

## 🏗️ Architecture

### Storage

Wallets are stored in-memory using a `HashMap` keyed by wallet name hash:

```rust
use wallet_adapter::WalletStorage;

let storage = WalletStorage::default();
let wallets = storage.get_wallets();
let phantom = storage.get_wallet("Phantom");
```

### Connection Info

Connection state is managed through `ConnectionInfo`:

```rust
let connection = adapter.connection_info().await;
let wallet = connection.connected_wallet()?;
let account = connection.connected_account()?;
```

### Error Handling

All operations return `WalletResult<T>`, which is `Result<T, WalletError>`. The error type provides detailed information about failures:

```rust
match adapter.connect_by_name("Phantom").await {
    Ok(_) => println!("Connected!"),
    Err(WalletError::WalletNotFound) => println!("Wallet not installed"),
    Err(WalletError::WalletConnectError(msg)) => println!("Connection failed: {}", msg),
    Err(e) => println!("Error: {}", e),
}
```

## 📦 Project Structure

```
.
├── crate/                    # Main library crate
│   └── src/
│       ├── adapter.rs       # WalletAdapter implementation
│       ├── errors.rs        # Error types
│       ├── events.rs        # Event handling
│       ├── storage.rs       # Wallet storage
│       ├── utils.rs         # Utility functions
│       └── wallet_ser_der/  # Wallet serialization/deserialization
├── templates/               # Framework templates
│   ├── dioxus-adapter/
│   ├── yew-adapter/
│   └── sycamore-adapter/
├── wallet-adapter-book/     # Documentation book
└── partial-idl-parser/      # IDL parser utility
```

## 🤝 Contributing

Contributions are welcome! Please read our contributing guidelines and code of conduct.

### Development Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/solana-wallet-adapter-rust.git
cd solana-wallet-adapter-rust

# Build the project
cargo build

# Run tests
cargo test

# Check code quality
cargo clippy
cargo fmt
```

### Code of Conduct

All contributors must agree to the [Rust Code of Conduct](https://www.rust-lang.org/policies/code-of-conduct).

## 📄 License

Licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE) or http://opensource.org/licenses/MIT)

at your option.

## 🙏 Acknowledgments

- Built for the Solana ecosystem
- Implements the [Wallet Standard](https://github.com/wallet-standard/wallet-standard)
- Inspired by the JavaScript wallet adapter ecosystem

## 📞 Contact & Support

- **Telegram**: [@moooncity](https://t.me/moooncity)
- **Issues**: [GitHub Issues](https://github.com/yourusername/solana-wallet-adapter-rust/issues)
- **Documentation**: [Full Book](https://jamiidao.github.io/SolanaWalletAdapter/)

---

<div align="center">

**Built with ❤️ for the Rust and Solana communities**

[⭐ Star us on GitHub](https://github.com/yourusername/solana-wallet-adapter-rust) • [📖 Read the Docs](https://docs.rs/wallet-adapter) • [💬 Get Support](https://t.me/moooncity)

</div>
