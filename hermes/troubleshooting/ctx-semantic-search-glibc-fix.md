---
type: Troubleshooting
title: "ctx CLI Semantic Search on WSL glibc 2.35 — ort-load-dynamic + ONNX Runtime"
description: "How to build ctx with ort-load-dynamic and provide libonnxruntime.so at runtime to enable semantic search on WSL Ubuntu 22.04 (glibc 2.35) without upgrading the OS."
tags: [ctx, semantic-search, onnx-runtime, glibc, wsl, rust, fastembed]
timestamp: 2026-07-10T15:55:00+09:00
---

# ctx CLI Semantic Search on WSL glibc 2.35

## Problem

The official `ctx` Linux x64 binary (v0.24.0) requires glibc 2.38+ (GLIBC_2.38, 2.39, 2.43 symbols). WSL Ubuntu 22.04 ships glibc 2.35, making the prebuilt binary unusable:

```
ctx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

Building from source with default features fails because `fastembed` pulls in `ort` (ONNX Runtime Rust bindings), which links against ONNX Runtime C++ libraries requiring `__isoc23_strtol` and other glibc 2.38+ symbols.

## Root Cause

- ctx uses `fastembed` (Rust crate) for local embedding generation via ONNX Runtime
- `ort-sys` (ONNX Runtime C++ bindings) statically links ONNX Runtime at build time
- The prebuilt ONNX Runtime binaries are compiled against a newer glibc
- `sqlite-vec` is compiled from C source locally (no glibc issue)

## Solution: `ort-load-dynamic` + Runtime ONNX Library

### Overview

1. Build ctx from source with `ort-load-dynamic` feature (defers ONNX linking to runtime via `dlopen`)
2. Download ONNX Runtime 1.24.2 Linux x64 from GitHub (requires only glibc 2.27 — compatible!)
3. Place `libonnxruntime.so` in `~/.local/lib/` and set `LD_LIBRARY_PATH`

### Step 1: Install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
source "$HOME/.cargo/env"
```

### Step 2: Clone ctx source

```bash
cd /tmp
git clone --depth 1 https://github.com/ctxrs/ctx.git ctx-src
cd ctx-src
```

### Step 3: Modify Cargo.toml

Edit `crates/ctx-cli/Cargo.toml`:

```toml
# Change fastembed features from ort-download-binaries-rustls-tls to ort-load-dynamic
fastembed = { version = "5.17.2", optional = true, default-features = false, features = ["hf-hub-rustls-tls", "ort-load-dynamic"] }
sqlite-vec = { version = "=0.1.9", optional = true }

# Add [features] section if not present
[features]
default = ["fastembed", "sqlite-vec"]
fastembed = ["dep:fastembed"]
sqlite-vec = ["dep:sqlite-vec"]
```

### Step 4: Verify build.rs cfg flags

`crates/ctx-cli/build.rs` should print the cfg flags for Linux x86_64:

```rust
// These lines should be active (not commented out):
println!("cargo:rustc-cfg=ctx_semantic_fastembed");
println!("cargo:rustc-cfg=ctx_sqlite_vec");
```

### Step 5: Build

```bash
cargo build --release -p ctx
```

Build succeeds without ONNX link errors because `ort-sys/disable-linking` is active via `load-dynamic`.

### Step 6: Install binary

```bash
cp /tmp/ctx-src/target/release/ctx ~/.local/bin/ctx
```

### Step 7: Download ONNX Runtime

```bash
cd /tmp
curl -fsSL -o onnxruntime.tgz \
  "https://github.com/microsoft/onnxruntime/releases/download/v1.24.2/onnxruntime-linux-x64-1.24.2.tgz"
tar xzf onnxruntime.tgz
```

### Step 8: Verify glibc compatibility

```bash
objdump -T onnxruntime-linux-x64-1.24.2/lib/libonnxruntime.so.1.24.2 | grep "GLIBC_" | sort -t_ -k2 -V | tail -5
# Max required: GLIBC_2.27 (expf, logf, powf) — glibc 2.35 is sufficient ✅

objdump -T onnxruntime-linux-x64-1.24.2/lib/libonnxruntime.so.1.24.2 | grep "GLIBCXX_" | sort -t_ -k2 -V | tail -5
# Max required: GLIBCXX_3.4.22 — Ubuntu 22.04 has up to 3.4.30 ✅
```

### Step 9: Install ONNX Runtime library

```bash
mkdir -p ~/.local/lib
cp /tmp/onnxruntime-linux-x64-1.24.2/lib/libonnxruntime.so.1.24.2 ~/.local/lib/
ln -sf ~/.local/lib/libonnxruntime.so.1.24.2 ~/.local/lib/libonnxruntime.so.1
ln -sf ~/.local/lib/libonnxruntime.so.1.24.2 ~/.local/lib/libonnxruntime.so
```

### Step 10: Set LD_LIBRARY_PATH

Add to `~/.bashrc`:

```bash
# ONNX Runtime for ctx semantic search
export LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
```

### Step 11: Configure ctx

`~/.ctx/config.toml`:

```toml
[analytics]
enabled = false

[search]
semantic = true

[daemon]
enabled = true
```

Also add analytics env vars to `~/.bashrc`:

```bash
export CTX_ANALYTICS_OFF=1
export CTX_DISABLE_ANALYTICS=1
export CTX_ANALYTICS_ENABLED=false
```

### Step 12: Initialize and index

```bash
export LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
export CTX_ANALYTICS_OFF=1 CTX_DISABLE_ANALYTICS=1 CTX_ANALYTICS_ENABLED=false

ctx setup --progress auto

# Start daemon for semantic indexing
ctx daemon run --idle-exit-seconds 300 &
# Wait for indexing to complete
ctx index wait --all
```

### Step 13: Verify

```bash
ctx status
# Expected: semantic_status: ready, semantic_embedded_items: 300+
```

## Search Backends

| Backend | Command | Description |
|---------|---------|-------------|
| lexical (FTS) | `ctx search "keyword" --backend lexical` | Fast text matching |
| semantic | `ctx search "concept" --backend semantic` | Vector similarity |
| hybrid (default) | `ctx search "keyword"` | FTS + semantic combined |

## Key Findings

- ONNX Runtime 1.24.2 Linux x64 requires only **glibc 2.27** (not 2.38+) — the glibc issue was in the **Rust ort-sys build-time linking**, not the ONNX library itself
- `ort-load-dynamic` uses `libloading` (dlopen) to load ONNX at runtime, avoiding build-time link errors
- The `all-MiniLM-L6-v2` embedding model (~90MB ONNX) is auto-downloaded from HuggingFace on first run
- sqlite-vec compiles from C source locally — no glibc issue
- Supported agent histories on this machine: Claude Code (`~/.claude/projects`), Hermes Agent (`~/.hermes/state.db`)

## Pitfalls

- **Daemon must inherit LD_LIBRARY_PATH**: The daemon process is spawned by ctx, so `LD_LIBRARY_PATH` must be set in the parent shell (`.bashrc`). If the daemon can't find `libonnxruntime.so`, it stays stuck at `semantic_index_status: acquiring_model`.
- **Stale daemon lock**: If daemon dies abnormally, remove `~/.ctx/daemon/daemon.lock` and `~/.ctx/semantic-worker.json` before restarting.
- **Don't use `--start-mode` or `--trigger-command`**: These are internal flags. Run `ctx daemon run --idle-exit-seconds 300` without them.
- **Cargo features must be explicitly declared**: `fastembed` and `sqlite-vec` are `optional = true`, so a `[features]` section with `default = ["fastembed", "sqlite-vec"]` is required for them to be compiled in.

## Environment

- WSL Ubuntu 22.04.5 LTS, glibc 2.35
- Rust 1.97.0 (stable)
- ctx 0.24.0 (source build)
- ONNX Runtime 1.24.2 (GitHub release binary)
- Build directory: `/tmp/ctx-src` (preserve for future rebuilds)

## Related

- [ctx GitHub Repository](https://github.com/ctxrs/ctx) — official source
- [ctx Documentation](https://ctx.rs/) — official docs
- [ort crate](https://crates.io/crates/ort) — ONNX Runtime Rust bindings
- [fastembed crate](https://crates.io/crates/fastembed) — local embedding generation