# Matrix Failure Audit

## Overview
This document audits the matrix CI workflow failures running across `ubuntu-latest` and `windows-latest` for Node.js versions 18, 20, 22, and 24 (experimental).

## Failure Analysis Grid

| Operating System | Node Version | Status | Failure Category | Step Failed | Root Cause |
|---|---|---|---|---|---|
| `ubuntu-latest` | 18 | PASS | N/A | N/A | N/A |
| `ubuntu-latest` | 20 | PASS | N/A | N/A | N/A |
| `ubuntu-latest` | 22 | FAIL | Runtime Version | `Run npm test` | `crypto.createCipher` API removed in Node 22 |
| `windows-latest` | 18 | FAIL | OS-Specific | `Run npm test` | Path concatenation using forward slash in `fileUtils.js` |
| `windows-latest` | 20 | FAIL | OS-Specific | `Run npm test` | Path concatenation using forward slash in `fileUtils.js` |
| `windows-latest` | 22 | FAIL | Combined OS & Runtime Version | `Run npm test` | Path concatenation + `crypto.createCipher` API removal |
| `ubuntu-latest` | 24 (exp) | FAIL | Runtime Version | `Run npm test` | `crypto.createCipher` API removed in Node 24 |

## Detailed Breakdown & Planned Fixes

### 1. Windows OS-Specific Failures (`windows-latest` on Node 18, 20, 22)
- **Step Name**: `Run npm test` (`src/fileUtils.test.js`)
- **Exact Error Line**:
  ```text
  Expected: "D:\\a\\matrix-debug-drill\\matrix-debug-drill\\src\\output\\report.txt"
  Received: "D:\\a\\matrix-debug-drill\\matrix-debug-drill\\src/output/report.txt"
  ```
- **Classification**: OS-Specific Failure (Path Separators)
- **Root Cause**: `getOutputPath` in `src/fileUtils.js` constructs paths using string concatenation with forward slashes (`__dirname + '/output/' + filename`), producing mismatched path separators (`\` vs `/`) on Windows. `readConfig` also uses string concatenation (`__dirname + '/configs/' + configName + '.json'`).
- **Planned Fix**:
  - Refactor `getOutputPath` and `readConfig` in `src/fileUtils.js` using Node's `path.join()`.
  - Add a `.gitattributes` file with `* text=auto eol=lf` to ensure cross-platform line ending normalization.

### 2. Node 22 & 24 Runtime Version Failures (`ubuntu-latest` & `windows-latest` on Node 22, 24)
- **Step Name**: `Run npm test` (`src/cryptoUtils.test.js`)
- **Exact Error Line**:
  ```text
  TypeError: crypto.createCipher is not a function
  ```
- **Classification**: Runtime Version Compatibility Failure
- **Root Cause**: `crypto.createCipher` and `crypto.createDecipher` were deprecated in Node 10 and completely removed in Node 22+.
- **Planned Fix**:
  - Update `encryptValue` and `decryptValue` in `src/cryptoUtils.js` to use `crypto.createCipheriv` and `crypto.createDecipheriv` with SHA-256 key derivation and random 16-byte initialization vectors (IVs).

### 3. Matrix Strategy & Experimental Execution
- **Configuration**:
  - Retain `fail-fast: false` so that matrix job execution continues when a single cell fails, ensuring complete cross-environment diagnostic visibility.
  - Retain `continue-on-error: ${{ matrix.experimental == true }}` so experimental Node 24 combinations provide test visibility without blocking pipeline status.
