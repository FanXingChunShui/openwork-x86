# OpenWrt Rust Build Fix

This directory contains patches for the OpenWrt Rust package to fix CI compatibility issues.

## Problem

The default Rust package in OpenWRT builds sets `llvm.download-ci-llvm = true` in the bootstrap configuration. However, Rust's bootstrap system rejects this setting on CI environments (detected via the `CI` environment variable), requiring the value to be `if-unchanged` instead.

Error:
```
thread 'main' panicked at src/bootstrap/src/core/config/config.rs:1681:21:
`llvm.download-ci-llvm` cannot be set to `true` on CI. Use `if-unchanged` instead.
```

## Solution

The patch files modify the Rust package's `config.toml.in` to use `if-unchanged` instead of `true` for:
- `llvm.download-ci-llvm`
- `download-ci-llvm` (in the `[llvm]` section)

Note: The Rust configuration format uses `if-unchanged` without quotes.

## Patch Files

### Primary Patch
- **`900-ci-llvm-download-fix.patch`**: Standard patch format with `a/` and `b/` prefixes (for `-p1` stripping, typically the default in OpenWrt)

### Alternative Patch  
- **`900-ci-llvm-download-fix-p0.patch`**: Alternative format without directory prefixes (for `-p0` stripping, if needed for certain OpenWrt/Rust versions)

## Application

These patches are automatically applied during the OpenWrt build process if placed in the correct directory structure:
- `feeds/packages/lang/rust/patches/`

The patches are copied into this location by the GitHub Actions workflow during the build.

## Troubleshooting

### Patch Application Failures

If the patch fails to apply with "can't find file to patch":

1. **Verify the rustc source structure**:
   - The `config.toml.in` file must exist in the root of the extracted rustc source
   - Different Rust versions may have different file structures

2. **Try alternative patch formats**:
   - Rename or copy `900-ci-llvm-download-fix-p0.patch` to be used instead
   - The workflow may need to specify a different patch application level

3. **Check the actual file content**:
   - Ensure the file starts with the expected comment: `# This is a configuration file intended for use on build systems`
   - Verify the exact format of the settings to be patched

4. **Use the workflow fallback**:
   - The GitHub Actions workflow includes direct sed patching as a fallback mechanism
   - This should catch and fix the config even if the patch file fails to apply

## Manual Application

If applying patches manually:

```bash
# Using patch command with -p1 (strip one directory level)
patch -p1 < 900-ci-llvm-download-fix.patch

# Or with -p0 (no stripping)
patch -p0 < 900-ci-llvm-download-fix-p0.patch
```

## Alternative Approach

If patch application fails consistently, use sed-based patching directly:
```bash
sed -i 's/llvm\.download-ci-llvm[[:space:]]*=[[:space:]]*true/llvm.download-ci-llvm = if-unchanged/g' config.toml.in
sed -i 's/download-ci-llvm[[:space:]]*=[[:space:]]*true/download-ci-llvm = if-unchanged/g' config.toml.in
```
