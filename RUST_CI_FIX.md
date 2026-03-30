# OpenWrt x86 Rust CI Build Fix - Summary

## Issue
The GitHub Actions workflow was failing with:
```
thread 'main' panicked at src/bootstrap/src/core/config/config.rs:1681:21:
`llvm.download-ci-llvm` cannot be set to `true` on CI. Use `if-unchanged` instead.
```

This occurred when building the Rust package for OpenWrt, as the default Rust package configuration had `llvm.download-ci-llvm = true`, which is incompatible with CI environments.

## Solution Overview

The fix implements a multi-layered approach to ensure the Rust package is properly configured:

### 1. Custom Patch File
**File**: `openwrt-patches/feeds/packages/lang/rust/900-ci-llvm-download-fix.patch`

This patch modifies the Rust package's `config.toml.in` to change:
- `llvm.download-ci-llvm = true` → `llvm.download-ci-llvm = if-unchanged`
- `download-ci-llvm = true` → `download-ci-llvm = if-unchanged`

Note: The Rust config format uses `if-unchanged` without quotes.

### 2. Workflow Modifications
**File**: `.github/workflows/build-openwrt.yml`

#### a) Enhanced Patch Step (`patch-rust`)
- Copies custom patches from repository to the build feeds directory
- Patches all patch files in the Rust feeds directory that contain the problematic setting
- Uses flexible sed patterns to handle various whitespace and quote formats

#### b) Verification Step (`verify-rust-patch`)
- Lists the contents of the patches directory
- Searches for any remaining `true` values (should be none)
- Searches for `if-unchanged` values (confirms patch was applied)

#### c) Enhanced Build Step (`build`)
- Sets `RUSTC_LLVM_DOWNLOAD=if-unchanged` environment variable
- Sets `RUST_BOOTSTRAP_CONFIG` with the proper configuration
- Exports `CARGO_BUILD_JOBS` setting

## How It Works

1. **During build preparation**:
   - OpenWrt source and feeds are cloned
   - Custom Rust patches are copied to the feeds directory
   - Both custom patches and any existing Rust patches are verified/patched to use `if-unchanged`

2. **During verification**:
   - The workflow checks that all patches have been properly updated
   - Provides diagnostic output showing what was patched

3. **During build**:
   - The Rust package is compiled with the corrected configuration
   - Environment variables provide additional safeguards
   - The bootstrap process now accepts the `if-unchanged` setting on CI

## Key Files Modified
- `.github/workflows/build-openwrt.yml` - Workflow with patch, verification, and direct patching steps
- `openwrt-patches/feeds/packages/lang/rust/900-ci-llvm-download-fix.patch` - Primary Rust package patch (with -p1 stripping)
- `openwrt-patches/feeds/packages/lang/rust/900-ci-llvm-download-fix-p0.patch` - Alternative patch format (with -p0 stripping)
- `openwrt-patches/feeds/packages/lang/rust/README.md` - Detailed patch documentation

## Patch Application Layers

The fix uses multiple layers to ensure the configuration is corrected:

1. **Patch File Copying** (`patch-rust` step):
   - Copies custom patches to `feeds/packages/lang/rust/patches/`
   - Patches the patch files themselves to ensure they have the correct format

2. **Patch Verification** (`verify-rust-patch` step):
   - Verifies patches contain `if-unchanged` values
   - Provides diagnostic output for troubleshooting

3. **Direct Config Patching** (`patch-rust-config-direct` step):
   - Fallback mechanism that directly patches any config files in the feeds directory
   - Ensures configuration is fixed even if the patch files don't apply

4. **Environment Variables** (in `build` step):
   - Sets `RUSTC_LLVM_DOWNLOAD=if-unchanged` for additional safeguards
   - Provides `RUST_BOOTSTRAP_CONFIG` environment variable

## Testing

When the workflow runs, it will:
1. Copy patches from the repository to the build environment
2. Patch the patch files if needed to ensure correct format
3. Verify patches are properly formatted
4. Apply direct config patches as a fallback mechanism
5. Build OpenWrt with the corrected Rust configuration
6. Generate firmware images successfully

## Troubleshooting Build Failures

If the build still fails with the Rust LLVM error:

1. **Check the GitHub Actions workflow output** for these key steps:
   - `patch-rust`: Should show "Patched" messages
   - `verify-rust-patch`: Should show patches were found and formatted
   - `patch-rust-config-direct`: Should show files were patched

2. **Debugging the patch application**:
   - Verify that `config.toml.in` exists in the extracted rustc source
   - Check if the rustc version changed and has a different file structure
   - Try using the alternative patch file (`900-ci-llvm-download-fix-p0.patch`)

3. **Advanced troubleshooting**:
   - Look for the actual error messages in the GitHub Actions logs
   - Search for "llvm.download-ci-llvm" in the build output
   - If direct patching still doesn't work, the file might be in a different location or format

4. **If patches don't work**:
   - Consider using environment variables or build-time configuration options
   - The Rust build system might need a `config.toml` file created or placed in a specific location
   - Check if the OpenWrt Rust package has its own mechanisms for configuration
