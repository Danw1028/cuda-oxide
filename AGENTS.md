# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

cuda-oxide is a custom `rustc` backend for compiling CUDA GPU kernels in pure Rust. The workspace has ~15 crates. See `README.md` for full setup instructions and `Justfile` for common commands.

### System dependencies

The following must be installed (the update script does NOT handle these — they are part of the VM snapshot):

- **Rust nightly** — pinned in `rust-toolchain.toml` (auto-installed by rustup)
- **CUDA Toolkit 13.0** — `cuda-nvcc-13-0`, `cuda-cudart-dev-13-0`, `libnvjitlink-13-0`, `libnvjitlink-dev-13-0`
- **LLVM 21** — from `apt.llvm.org` (Noble/llvm-toolchain-noble-21). Provides `llc-21` with NVPTX backend.
- **Clang 21** — `clang-21`, `libclang-common-21-dev` (for `bindgen` in `cuda-bindings`)
- **just** — task runner (installed to `/usr/local/bin`)

### Environment variables

These must be set before builds/tests (added to `~/.bashrc`):

```bash
export PATH="/usr/local/cuda/bin:$PATH"
export CUDA_TOOLKIT_PATH="/usr/local/cuda"
SYSROOT="$(rustc --print sysroot 2>/dev/null)"
export LIBRARY_PATH="${SYSROOT}/lib:${LIBRARY_PATH:-}"
export LD_LIBRARY_PATH="${SYSROOT}/lib:${LD_LIBRARY_PATH:-}"
```

The `LIBRARY_PATH`/`LD_LIBRARY_PATH` pointing at the rustc sysroot `lib/` is **required** for crates that use `rustc_private` (the compiler dialect crates link against `librustc_driver`).

### Running without a GPU

This Cloud VM does **not** have an NVIDIA GPU. The following work without a GPU:

| Command | What it does |
|---|---|
| `just clippy` | Lint (needs CUDA headers, not GPU) |
| `just test` | Unit tests for dialect/macro/host crates |
| `just fmt-check` | Format check |
| `cargo oxide doctor` | Validates toolchain setup (needs `libcuda.so.1` stub — symlink at `/usr/local/cuda/lib64/stubs/libcuda.so.1`) |
| `cargo oxide build <example>` | Full Rust→PTX compilation (needs stubs + `LD_LIBRARY_PATH` including stubs dir) |

To run `cargo oxide build` or `cargo oxide doctor`, include the CUDA stubs directory in library paths:

```bash
export LD_LIBRARY_PATH="/usr/local/cuda/lib64/stubs:/usr/local/cuda/lib64:${LD_LIBRARY_PATH}"
export LIBRARY_PATH="/usr/local/cuda/lib64/stubs:/usr/local/cuda/lib64:${LIBRARY_PATH}"
```

The following **require a GPU** and will fail in this environment: `cargo oxide run <example>`, `scripts/smoketest.sh`.

### Key gotchas

- The `rustc-codegen-cuda` crate is **not** a workspace member. It has its own `Cargo.toml` and is built separately by `cargo oxide`. Do not try to `cargo build -p rustc-codegen-cuda`.
- `cargo clippy --workspace` requires CUDA toolkit headers (for `cuda-bindings` bindgen). The CI workflow sets `CUDA_TOOLKIT_PATH`.
- Unit tests in CI are run with `LIBRARY_PATH` and `LD_LIBRARY_PATH` pointing at the rustc sysroot `lib/` directory. Without this, linking fails for `rustc_private` crates.
