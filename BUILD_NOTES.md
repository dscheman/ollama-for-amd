# Ollama for AMD - Custom Build Notes

## Build Date
November 28, 2025

## Version
v0.12.7 (Branch_v0.12.7)

## Target GPU
AMD Radeon gfx1103 (Radeon 780M APU)

## Prerequisites

### Required Software
- **ROCm 6.4** with custom gfx1103 support
  - **Installation Path:** `C:\Program Files\AMD\ROCm\6.4\`
  - **Environment Variable:** `HIP_PATH=C:\Program Files\AMD\ROCm\6.4`
  - **Source:** Custom build from https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU
  - **Status:** ✓ Version 6.4 is already installed and configured (no additional action needed)
  - **Critical Files:**
    - `rocblas.dll` with gfx1103 support at `C:\Program Files\AMD\ROCm\6.4\bin\`
    - Tensile library files in `C:\Program Files\AMD\ROCm\6.4\bin\rocblas\library\`

- **Visual Studio 2022** with C++ development tools
- **CMake** (version 3.21 or later)
- **Ninja** build system (usually in Visual Studio installation)
- **Go compiler** (for building Ollama executables)
- **MinGW64** (for C/C++ compilation with Go)
- **PowerShell** with execution policy allowing scripts

### Environment Variables
- `HIP_PATH=C:\Program Files\AMD\ROCm\6.4` - Points to ROCm installation (required)
- `CGO_ENABLED=1` - Required for Go builds with C dependencies

## Critical Build Requirements

### Required Components (Build Order Matters!)

To successfully build Ollama with ROCm GPU support, you MUST build these components in order:

1. **buildCPU** - REQUIRED FIRST
   - Builds `ggml-base.dll` - the foundation library that ALL backends depend on
   - Builds CPU variant DLLs (`ggml-cpu-*.dll`)
   - Without this, GPU will NOT be detected even if ROCm is built correctly

2. **buildROCm** - GPU Support
   - Builds `ggml-hip.dll` with GPU kernels
   - Installs ROCm libraries to `lib/ollama/rocm/`

3. **buildOllama** - CLI executable
   - Builds `ollama.exe`

4. **buildApp** - GUI executable (optional)
   - Builds the GUI application

5. **gatherDependencies** - Runtime DLLs
   - Copies Microsoft Visual C++ runtime DLLs
   - Copies LLVM CRT DLLs (api-ms-win-crt-*.dll)

### Complete Build Command

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\build_windows.ps1 buildCPU buildROCm buildOllama buildApp gatherDependencies
```

### Alternative: Full Build (includes CUDA/Vulkan if available)
```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\build_windows.ps1
```

## Configuration Changes Made

### 1. CMakePresets.json (Line 71)
**Original:**
```json
"AMDGPU_TARGETS": "gfx940;gfx941;gfx942;gfx1010;gfx1012;gfx1030;gfx1100;gfx1101;gfx1102;gfx1103;gfx1151;gfx1200;gfx1201;gfx908:xnack-;gfx90a:xnack+;gfx90a:xnack-"
```

**Modified (for gfx1103 only build):**
```json
"AMDGPU_TARGETS": "gfx1103"
```

**Note:** This reduces the size of compiled GPU kernels but gfx1103 support is fully functional.

### 2. CMakeLists.txt (Line 104)
**Original:**
```cmake
"^gfx(803|90[012]|906(:xnack-)|90c(:xnack-)|1010(:xnack-)|1011(:xnack-)|1012(:xnack-)|103[0-6]|110[0-3]|115[0123]|120[01])$"
```

**Modified (for gfx1103 only build):**
```cmake
"^gfx1103$"
```

**Purpose:** Filters which GPU kernel files are included in the final distribution.

### 3. build_windows.ps1 (Line 182)
**Note:** Script already includes a cleanup command to remove gfx906 files after ROCm build:
```powershell
Remove-Item -Path $script:DIST_DIR\lib\ollama\rocm\rocblas\library\*gfx906* -ErrorAction SilentlyContinue
```

## Common Issues & Solutions

### Issue: GPU Not Detected (Exit Code 1)
**Cause:** Missing `ggml-base.dll` or CPU backend DLLs

**Solution:** Always run `buildCPU` first! This builds the foundation library that all backends depend on.

**Verification:**
```powershell
Get-ChildItem .\dist\windows-amd64\lib\ollama -File | Select-Object Name, Length
```

Should show at least:
- `ggml-base.dll` (~767KB)
- `ggml-cpu-*.dll` (multiple variants)
- `msvcp140*.dll` and `vcruntime140*.dll`
- `api-ms-win-crt-*.dll` (10 files)

### Issue: Small ggml-hip.dll (~40MB instead of ~744MB)
**Expected for gfx1103-only build:** Yes, when targeting only gfx1103, the DLL is smaller because it contains fewer GPU kernels. This is normal and correct.

**Full GPU support DLL:** ~744MB (includes all GPU architectures)
**gfx1103-only DLL:** ~40MB (only gfx1103 kernels)

### Issue: Build Fails on ROCm Step
**Check:** 
1. ROCm is installed and `%HIP_PATH%` is set
2. Ninja is in PATH
3. Clang compiler is accessible

## Clean Rebuild Steps

```powershell
# 1. Clean previous build
Remove-Item -Recurse -Force .\dist\* -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force .\build\* -ErrorAction SilentlyContinue

# 2. Regenerate Go files
$env:CGO_ENABLED="1"; go generate ./...

# 3. Build all required components
powershell -ExecutionPolicy Bypass -File .\scripts\build_windows.ps1 buildCPU buildROCm buildOllama buildApp gatherDependencies

# 4. Test
cd .\dist\windows-amd64
.\ollama.exe --version
```

## Output Structure

```
dist/windows-amd64/
├── ollama.exe                              # Main CLI executable
├── windows-amd64-app.exe                   # GUI executable
└── lib/ollama/
    ├── ggml-base.dll                       # REQUIRED - Base library
    ├── ggml-cpu-*.dll                      # CPU backend variants (8 files)
    ├── ggml-hip.dll                        # CRITICAL - Fails without base!
    ├── msvcp140*.dll                       # MSVC runtime (4 files)
    ├── vcruntime140*.dll                   # VC runtime (2 files)
    ├── api-ms-win-crt-*.dll               # C runtime (10 files)
    └── rocm/
        ├── ggml-hip.dll                    # ROCm backend
        └── rocblas/library/
            ├── *gfx1103* files             # GPU kernels for gfx1103
            └── TensileLibrary_*.dat        # Tensile library data
```

## Key Lessons Learned

1. **ggml-base.dll is absolutely critical** - Without it, no backend (CPU, GPU, etc.) will work
2. **Build order matters** - CPU must be built before testing GPU functionality
3. **Component installation** - CMake installs different components (CPU, HIP, CUDA) separately
4. **Smaller DLL is OK** - When targeting single GPU, DLL size reduction is expected and normal

## Future Upgrades

When upgrading to a new version:

1. Check if `CMakePresets.json` AMDGPU_TARGETS has changed
2. Check if `CMakeLists.txt` GPU regex pattern has changed
3. Reapply gfx1103-only modifications if desired
4. ALWAYS build in this order: CPU → ROCm → Ollama → App → Dependencies
5. Test with `ollama.exe --version` and `ollama.exe serve` before considering it complete
