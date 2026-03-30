# Rust CI Build Fix - Implementation Summary

## Date: March 30, 2026
## Issue: Patch file failing to apply during rustc compilation in CI

### Problem Analyzed

The GitHub Actions workflow was failing with:
```
Applying /home/runner/work/openwrt-x86/openwrt-x86/openwrt-src/feeds/packages/lang/rust/patches/900-ci-llvm-download-fix.patch using plaintext:
can't find file to patch at input line 3
Perhaps you used the wrong -p or --strip option?
...
Patch failed!
```

Root Cause: The patch file was attempting to modify `config.toml.in`, likely with incorrect paths, context, or the file structure not matching what was expected.

### Solutions Implemented

#### 1. Patch File Corrections
- **Updated** `900-ci-llvm-download-fix.patch`: 
  - Fixed format to properly target `config.toml.in`
  - Changed values to not use quotes: `if-unchanged` instead of `"if-unchanged"`
  - Maintained proper context lines for patch matching
  - Format: Standard unified diff with `a/` and `b/` prefixes (for `-p1` stripping)

- **Created** `900-ci-llvm-download-fix-p0.patch`:
  - Alternative format without directory prefixes
  - Targets systems that might use `-p0` stripping
  - Provides compatibility if OpenWrt uses different patching mechanisms

#### 2. Workflow Enhancements (`.github/workflows/build-openwrt.yml`)

**Step: patch-rust**
- Fixed sed patterns to match the correct value format (without quotes)
- Patterns now correctly handle both formats:
  - `llvm.download-ci-llvm = true` → `llvm.download-ci-llvm = if-unchanged`
  - `download-ci-llvm = true` → `download-ci-llvm = if-unchanged`

**Step: verify-rust-patch** 
- Updated to verify patches use correct format (`if-unchanged` without quotes)
- Provides diagnostic output for troubleshooting

**New Step: patch-rust-config-direct** (Fallback Mechanism)
- Directly patches any `config.toml*` files found in `feeds/packages/lang/rust/`
- Uses sed to apply the same fixes regardless of patch file success
- Ensures configuration is corrected even if patch files fail to apply
- Multiple sed patterns handle different quote variations

#### 3. Documentation Updates

**Updated** `openwrt-patches/feeds/packages/lang/rust/README.md`:
- Clarified that values should NOT have quotes
- Documented both patch file options and their purposes
- Added troubleshooting section with multiple approaches
- Included manual patch application instructions
- Provided sed command alternatives

**Updated** `RUST_CI_FIX.md`:
- Added information about multiple patch files
- Described multi-layer fix approach
- Enhanced troubleshooting guide with specific steps
- Documented what to look for in GitHub Actions logs
- Added advanced troubleshooting for persistent failures

### Multi-Layer Fix Architecture

The solution now uses four layers to ensure the Rust CI configuration is corrected:

1. **Patch File Application**: Standard patch mechanism through OpenWrt's build system
2. **Pre-Build Verification**: Checks that patches are correctly formatted
3. **Fallback Direct Patching**: Patches config files if patch mechanism fails
4. **Environment Variables**: Additional safeguards in the build step

### Testing Recommendations

When the fixed workflow runs:

1. Check `patch-rust` step output for successful patch copying/formatting
2. Review `verify-rust-patch` step to confirm `if-unchanged` values are present
3. Monitor `patch-rust-config-direct` step for fallback mechanism status
4. Watch the `build` step output for the Rust compilation
5. Look for successful firmware generation if everything works

### Fallback Procedures

If the build still fails:

1. **First attempt**: Run workflow as-is with current fixes
2. **If patch fails**: Fallback direct patching should handle it
3. **If direct patching fails**: 
   - Check GitHub Actions logs for actual file paths
   - Verify `config.toml.in` exists in extracted rustc source
   - Consider if Rust version has different structure
4. **Last resort**: Manually edit patch paths or use environment-variable-only approach

### Files Modified

- ✅ `openwrt-patches/feeds/packages/lang/rust/900-ci-llvm-download-fix.patch` - Fixed
- ✅ `openwrt-patches/feeds/packages/lang/rust/900-ci-llvm-download-fix-p0.patch` - Created
- ✅ `openwrt-patches/feeds/packages/lang/rust/README.md` - Enhanced
- ✅ `.github/workflows/build-openwrt.yml` - Updated sed patterns and added fallback mechanism
- ✅ `RUST_CI_FIX.md` - Enhanced documentation

### Verification Checklist

- [x] Patch files have correct format (no quotes in values)
- [x] Both `-p1` and `-p0` variants provided
- [x] Workflow sed patterns updated to match new value format
- [x] Fallback direct patching mechanism added
- [x] Documentation updated with all changes
- [x] Troubleshooting guides enhanced
- [x] Alternative patch approaches documented

### Next Steps

1. Run the GitHub Actions workflow with these fixes
2. Monitor the workflow logs for success indicators
3. If successful, firmware should build without Rust LLVM errors
4. If issues persist, refer to the enhanced troubleshooting guide in RUST_CI_FIX.md
