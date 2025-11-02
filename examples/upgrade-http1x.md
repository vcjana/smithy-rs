# Upgrading from http@0.2/hyper@0.14 to http@1.x/hyper@1.x

This guide provides a comprehensive walkthrough for upgrading your smithy-rs server applications from http@0.2.x and hyper@0.14 to http@1.x and hyper@1.x.

## Table of Contents

- [Overview](#overview)
- [Why Upgrade?](#why-upgrade)
- [Before You Begin](#before-you-begin)
- [Dependency Updates](#dependency-updates)
- [Server-Side Changes](#server-side-changes)
- [Client-Side Changes](#client-side-changes)
- [Test Infrastructure Updates](#test-infrastructure-updates)
- [Common Migration Patterns](#common-migration-patterns)
- [Troubleshooting](#troubleshooting)
- [Migration Checklist](#migration-checklist)

## Overview

The http and hyper crates have released major version updates (http@1.x and hyper@1.x) with significant API improvements and breaking changes. This guide helps you migrate your smithy-rs server applications to these new versions.

**Key Changes:**
- http: 0.2.x → 1.x
- hyper: 0.14.x → 1.x
- New hyper-util crate for additional utilities
- aws-smithy-http-server replaces aws-smithy-legacy-http-server

## Why Upgrade?

- **Better Performance**: Hyper 1.x has improved performance and reduced allocations
- **Improved API**: More ergonomic and safer APIs in both http and hyper
- **Active Support**: Future updates and bug fixes will target 1.x versions
- **Ecosystem Alignment**: New libraries are targeting http@1.x and hyper@1.x
- **Security Updates**: Continued security patches for 1.x line

## Before You Begin

**Important Considerations:**

1. **Breaking Changes**: This is a major version upgrade with breaking API changes
2. **Testing Required**: Thoroughly test your application after migration
3. **Gradual Migration**: Consider migrating one service at a time
4. **Legacy Examples**: The `examples/legacy/` directory contains fully working http@0.2 examples for reference

**Compatibility:**
- Minimum Rust version: Check your generated SDK's rust-toolchain.toml
- Tokio runtime: Still compatible with tokio 1.x
- Tower middleware: Compatible with tower 0.4

## Dependency Updates

### Cargo.toml Changes

#### Server Dependencies

**Before (http@0.2/hyper@0.14):**
```toml
[dependencies]
http = "0.2"
hyper = { version = "0.14.26", features = ["server"] }
tokio = "1.26.0"
tower = "0.4"

# Server SDK with legacy HTTP support
pokemon-service-server-sdk = {
    path = "../pokemon-service-server-sdk/",
    package = "pokemon-service-server-sdk-http0x",
    features = ["request-id"]
}

[dev-dependencies]
hyper = { version = "0.14.26", features = ["server", "client"] }
hyper-rustls = { version = "0.24", features = ["http2"] }
aws-smithy-legacy-http = { path = "../../../rust-runtime/aws-smithy-legacy-http/" }
```

**After (http@1.x/hyper@1.x):**
```toml
[dependencies]
http = "1"
hyper = { version = "1", features = ["server"] }
hyper-util = { version = "0.1", features = ["tokio", "server", "server-auto", "service"] }
tokio = { version = "1.26.0", features = ["rt-multi-thread", "macros"] }
tower = "0.4"

# Server SDK with http@1.x support
pokemon-service-server-sdk = {
    path = "../pokemon-service-server-sdk/",
    features = ["request-id"]
}

[dev-dependencies]
bytes = "1"
http-body-util = "0.1"
hyper = { version = "1", features = ["server", "client"] }
hyper-util = { version = "0.1", features = ["client", "client-legacy", "http1", "http2"] }
hyper-rustls = { version = "0.27", features = ["http2"] }
aws-smithy-runtime = { path = "../../rust-runtime/aws-smithy-runtime" }
```

**Key Changes:**
- `http`: `0.2` → `1`
- `hyper`: `0.14.26` → `1`
- **New**: `hyper-util` crate for server and client utilities
- **New**: `bytes` and `http-body-util` for body handling
- `hyper-rustls`: `0.24` → `0.27`
- Server SDK package name: no longer needs `-http0x` suffix
- Replace `aws-smithy-legacy-http` with `aws-smithy-runtime`

#### Client Dependencies

**Before:**
```toml
[dependencies]
aws-smithy-runtime = { version = "1.0", features = ["client"] }
aws-smithy-runtime-api = "1.0"
```

**After:**
```toml
[dependencies]
aws-smithy-http-client = "1.0"  # Replaces direct hyper usage
aws-smithy-runtime = { version = "1.0", features = ["client"] }
aws-smithy-runtime-api = "1.0"
```

## Server-Side Changes

### 1. Import Changes

**Before:**
```rust
use hyper::StatusCode;
```

**After:**
```rust
use http::StatusCode;
use pokemon_service_server_sdk::server::serve;
use tokio::net::TcpListener;
```

**Key Points:**
- `StatusCode` and other HTTP types now come from the `http` crate, not `hyper`
- Import the `serve` helper from your server SDK
- Use `tokio::net::TcpListener` instead of hyper's built-in listener

### 2. Server Initialization

**Before (hyper@0.14):**
```rust
#[tokio::main]
pub async fn main() {
    // ... setup config and app ...

    let make_app = app.into_make_service_with_connect_info::<SocketAddr>();

    let bind: SocketAddr = format!("{}:{}", args.address, args.port)
        .parse()
        .expect("unable to parse the server bind address and port");

    let server = hyper::Server::bind(&bind).serve(make_app);

    if let Err(err) = server.await {
        eprintln!("server error: {err}");
    }
}
```

**After (hyper@1.x):**
```rust
#[tokio::main]
pub async fn main() {
    // ... setup config and app ...

    let make_app = app.into_make_service_with_connect_info::<SocketAddr>();

    let bind: SocketAddr = format!("{}:{}", args.address, args.port)
        .parse()
        .expect("unable to parse the server bind address and port");

    let listener = TcpListener::bind(bind)
        .await
        .expect("failed to bind TCP listener");

    // Optional: Get the actual bound address (useful for port 0)
    let actual_addr = listener.local_addr().expect("failed to get local address");
    eprintln!("Server listening on {}", actual_addr);

    if let Err(err) = serve(listener, make_app).await {
        eprintln!("server error: {err}");
    }
}
```

**Key Changes:**
1. Replace `hyper::Server::bind(&bind)` with `TcpListener::bind(bind).await`
2. Use the `serve()` helper function instead of `.serve(make_app)`
3. TcpListener binding is now async (requires `.await`)
4. Can get actual bound address with `.local_addr()` (useful for testing with port 0)

### 3. Service Building

The service building API remains the same:

```rust
let app = PokemonService::builder(config)
    .get_pokemon_species(get_pokemon_species)
    .get_storage(get_storage_with_local_approved)
    .get_server_statistics(get_server_statistics)
    .capture_pokemon(capture_pokemon)
    .do_nothing(do_nothing_but_log_request_ids)
    .check_health(check_health)
    .build()
    .expect("failed to build an instance of PokemonService");

let make_app = app.into_make_service_with_connect_info::<SocketAddr>();
```

No changes needed here! The service builder API is stable.

## Client-Side Changes

### HTTP Client Connector Setup

**Before (hyper@0.14 with hyper-rustls):**
```rust
use aws_smithy_runtime::client::http::hyper_014::HyperClientBuilder;
use hyper_rustls::ConfigBuilderExt;

fn create_client() -> PokemonClient {
    let tls_config = rustls::ClientConfig::builder()
        .with_safe_defaults()
        .with_native_roots()
        .with_no_client_auth();

    let tls_connector = hyper_rustls::HttpsConnectorBuilder::new()
        .with_tls_config(tls_config)
        .https_or_http()
        .enable_http1()
        .enable_http2()
        .build();

    let http_client = HyperClientBuilder::new().build(tls_connector);

    let config = pokemon_service_client::Config::builder()
        .endpoint_url(POKEMON_SERVICE_URL)
        .http_client(http_client)
        .build();

    pokemon_service_client::Client::from_conf(config)
}
```

**After (http@1.x with aws-smithy-http-client):**
```rust
use aws_smithy_http_client::{Builder, tls};

fn create_client() -> PokemonClient {
    // Create a TLS context with platform trusted root certificates
    let tls_context = tls::TlsContext::builder()
        .with_trust_store(tls::TrustStore::default())
        .build()
        .expect("failed to build TLS context");

    // Create an HTTP client using rustls with AWS-LC crypto provider
    let http_client = Builder::new()
        .tls_provider(tls::Provider::Rustls(tls::rustls_provider::CryptoMode::AwsLc))
        .tls_context(tls_context)
        .build_https();

    let config = pokemon_service_client::Config::builder()
        .endpoint_url(POKEMON_SERVICE_URL)
        .http_client(http_client)
        .build();

    pokemon_service_client::Client::from_conf(config)
}
```

**Key Changes:**
1. Replace `HyperClientBuilder` with `aws_smithy_http_client::Builder`
2. Use `tls::TlsContext` and `tls::TrustStore` instead of direct rustls config
3. Specify TLS provider explicitly (Rustls with AWS-LC or Ring crypto)
4. Use `.build_https()` instead of passing a connector
5. Much simpler API with better defaults

### Using Default Client

If you don't need custom TLS configuration, you can use the default client:

```rust
// Default client with system trust store
let config = pokemon_service_client::Config::builder()
    .endpoint_url(POKEMON_SERVICE_URL)
    .build();

let client = pokemon_service_client::Client::from_conf(config);
```

## Test Infrastructure Updates

### Test Helpers

**Before (http@0.14):**
```rust
use std::{process::Command, time::Duration};
use tokio::time::sleep;

pub async fn run_server() -> ChildDrop {
    let crate_name = std::env::var("CARGO_PKG_NAME").unwrap();
    let child = Command::cargo_bin(crate_name).unwrap().spawn().unwrap();
    sleep(Duration::from_millis(500)).await;
    ChildDrop(child)
}

pub fn base_url() -> String {
    format!("http://{DEFAULT_ADDRESS}:{DEFAULT_PORT}")
}

pub fn client() -> Client {
    let config = Config::builder()
        .endpoint_url(format!("http://{DEFAULT_ADDRESS}:{DEFAULT_PORT}"))
        .build();
    Client::from_conf(config)
}
```

**After (http@1.x with dynamic port detection):**
```rust
use std::{
    io::{BufRead, BufReader},
    process::{Command, Stdio},
    time::Duration,
};
use tokio::time::timeout;

pub struct ServerHandle {
    pub child: ChildDrop,
    pub port: u16,
}

pub async fn run_server() -> ServerHandle {
    let mut child = Command::new(assert_cmd::cargo::cargo_bin!("pokemon-service"))
        .args(["--port", "0"]) // Use port 0 for random available port
        .stderr(Stdio::piped())
        .spawn()
        .unwrap();

    // Wait for the server to signal it's ready by reading stderr
    let stderr = child.stderr.take().unwrap();
    let ready_signal = tokio::task::spawn_blocking(move || {
        let reader = BufReader::new(stderr);
        for line in reader.lines() {
            if let Ok(line) = line {
                if let Some(port_str) = line.strip_prefix("SERVER_READY:") {
                    if let Ok(port) = port_str.parse::<u16>() {
                        return Some(port);
                    }
                }
            }
        }
        None
    });

    // Wait for the ready signal with a timeout
    let port = match timeout(Duration::from_secs(5), ready_signal).await {
        Ok(Ok(Some(port))) => port,
        _ => panic!("Server did not become ready within 5 seconds"),
    };

    ServerHandle {
        child: ChildDrop(child),
        port,
    }
}

pub fn base_url(port: u16) -> String {
    format!("http://{DEFAULT_ADDRESS}:{port}")
}

pub fn client(port: u16) -> Client {
    let config = Config::builder()
        .endpoint_url(format!("http://{DEFAULT_ADDRESS}:{port}"))
        .build();
    Client::from_conf(config)
}
```

**Key Improvements:**
1. **Dynamic port allocation**: Use port 0 to get a random available port
2. **Server ready signaling**: Wait for server to emit "SERVER_READY:{port}" before running tests
3. **No more sleep()**: Deterministic server startup detection
4. **Better test isolation**: Each test can run on its own port
5. **Timeout protection**: Fail fast if server doesn't start

**Update your server to emit the ready signal:**
```rust
let listener = TcpListener::bind(bind).await.expect("failed to bind TCP listener");
let actual_addr = listener.local_addr().expect("failed to get local address");
eprintln!("SERVER_READY:{}", actual_addr.port());
```

### Test Usage

**Before:**
```rust
#[tokio::test]
async fn test_get_pokemon_species() {
    let _child = common::run_server().await;
    let client = common::client();
    // ... test code ...
}
```

**After:**
```rust
#[tokio::test]
async fn test_get_pokemon_species() {
    let server = common::run_server().await;
    let client = common::client(server.port);
    // ... test code ...
}
```

## Common Migration Patterns

### 1. StatusCode Usage

**Before:**
```rust
use hyper::StatusCode;

let status = StatusCode::OK;
let status = StatusCode::NOT_FOUND;
```

**After:**
```rust
use http::StatusCode;

let status = StatusCode::OK;
let status = StatusCode::NOT_FOUND;
```

### 2. Response Building

**Before:**
```rust
use hyper::{Body, Response, StatusCode};

let response = Response::builder()
    .status(StatusCode::OK)
    .body(Body::from("Hello"))
    .unwrap();
```

**After:**
```rust
use http::{Response, StatusCode};
use http_body_util::Full;
use bytes::Bytes;

let response = Response::builder()
    .status(StatusCode::OK)
    .body(Full::new(Bytes::from("Hello")))
    .unwrap();
```

### 3. Request Handling

Most request handling code remains the same thanks to smithy-rs abstractions:

```rust
// This works the same in both versions
pub async fn get_pokemon_species(
    input: GetPokemonSpeciesInput,
) -> Result<GetPokemonSpeciesOutput, GetPokemonSpeciesError> {
    // Your handler code
}
```

### 4. Middleware and Layers

Tower layers continue to work the same way:

```rust
// No changes needed for tower layers
let config = PokemonServiceConfig::builder()
    .layer(AddExtensionLayer::new(Arc::new(State::default())))
    .layer(AlbHealthCheckLayer::from_handler("/ping", |_req| async {
        StatusCode::OK
    }))
    .layer(ServerRequestIdProviderLayer::new())
    .build();
```

## Troubleshooting

### Common Errors and Solutions

#### 1. "no method named `serve` found for struct `hyper::Server`"

**Error:**
```
error[E0599]: no method named `serve` found for struct `hyper::Server` in the current scope
```

**Solution:**
Use `TcpListener` with the `serve()` function instead:
```rust
let listener = TcpListener::bind(bind).await?;
serve(listener, make_app).await?;
```

#### 2. "the trait `tower::Service` is not implemented"

**Error:**
```
error[E0277]: the trait `tower::Service<Request<Incoming>>` is not implemented for `...`
```

**Solution:**
Make sure you're using `hyper-util` with the right features:
```toml
hyper-util = { version = "0.1", features = ["tokio", "server", "server-auto", "service"] }
```

#### 3. "cannot find `HyperClientBuilder` in module"

**Error:**
```
error[E0432]: unresolved import `aws_smithy_runtime::client::http::hyper_014::HyperClientBuilder`
```

**Solution:**
Use the new client builder:
```rust
use aws_smithy_http_client::Builder;

let http_client = Builder::new()
    .build_https();
```

#### 4. Type mismatch with `Body` or `Bytes`

**Error:**
```
error[E0308]: mismatched types
expected struct `hyper::body::Incoming`
found struct `http_body_util::combinators::boxed::UnsyncBoxBody<...>`
```

**Solution:**
Add `http-body-util` and `bytes` dependencies:
```toml
bytes = "1"
http-body-util = "0.1"
```

Then use the appropriate body types from `http-body-util`.

#### 5. Tests failing with "connection refused"

**Problem:**
Tests start before server is ready.

**Solution:**
Implement the server ready signaling pattern shown in [Test Infrastructure Updates](#test-infrastructure-updates).

### Getting Help

- **Examples**: Check the `examples/` directory for working http@1.x code
- **Legacy Examples**: Check `examples/legacy/` for http@0.2 reference
- **Documentation**: https://docs.rs/aws-smithy-http-server/
- **GitHub Issues**: https://github.com/smithy-lang/smithy-rs/issues

## Migration Checklist

Use this checklist to track your migration progress:

### Pre-Migration
- [ ] Review this guide completely
- [ ] Backup your current working code
- [ ] Review the legacy examples in `examples/legacy/`
- [ ] Identify all places using http/hyper directly
- [ ] Plan for testing time

### Dependencies
- [ ] Update `http` from `0.2` to `1`
- [ ] Update `hyper` from `0.14` to `1`
- [ ] Add `hyper-util` with required features
- [ ] Add `bytes` and `http-body-util` if needed
- [ ] Update `hyper-rustls` if used (0.24 → 0.27)
- [ ] Update server SDK package (remove `-http0x` suffix)
- [ ] Remove `aws-smithy-legacy-http` references
- [ ] Add `aws-smithy-runtime` if needed

### Server Code
- [ ] Change `use hyper::StatusCode` to `use http::StatusCode`
- [ ] Import `serve` from server SDK
- [ ] Import `tokio::net::TcpListener`
- [ ] Replace `hyper::Server::bind()` with `TcpListener::bind().await`
- [ ] Replace `.serve(make_app)` with `serve(listener, make_app).await`
- [ ] Add server ready signaling (optional but recommended)

### Client Code
- [ ] Replace `HyperClientBuilder` with `aws_smithy_http_client::Builder`
- [ ] Update TLS configuration to use new API
- [ ] Test custom connector setups

### Test Code
- [ ] Update test helpers for dynamic port allocation
- [ ] Implement server ready detection
- [ ] Update all test usages to pass port parameter
- [ ] Remove arbitrary sleep() calls

### Validation
- [ ] Run `cargo check` - fix compilation errors
- [ ] Run `cargo test` - ensure all tests pass
- [ ] Run `cargo clippy` - address warnings
- [ ] Manual testing of all endpoints
- [ ] Load/performance testing if applicable
- [ ] Review security configurations (TLS, auth)

### Documentation
- [ ] Update README with new dependency versions
- [ ] Update code examples in documentation
- [ ] Add migration notes for your team
- [ ] Update deployment scripts if needed

### Deployment
- [ ] Test in development environment
- [ ] Test in staging environment
- [ ] Plan rollback strategy
- [ ] Deploy to production
- [ ] Monitor for issues

## Conclusion

Migrating from http@0.2/hyper@0.14 to http@1.x/hyper@1.x involves updating dependencies and making targeted changes to server initialization and client setup code. The smithy-rs abstractions shield you from many breaking changes, making the migration more straightforward than a raw hyper upgrade.

The new versions offer improved performance, better APIs, and are the future of the Rust HTTP ecosystem. While migration requires effort, the benefits make it worthwhile.

For additional help, consult the working examples in the `examples/` directory or reach out through GitHub issues.
