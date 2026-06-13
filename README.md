# MiKTeX - Native Port for Windows on ARM64 (WOA)

**English** | [中文说明](./README.zh-CN.md)

This project is a fork of the official [MiKTeX](https://github.com/MiKTeX/miktex) (v26.5 / `build-26.5` branch) tailored specifically for **Windows on ARM64 (WOA)**. By removing heavy Qt/MFC graphical interface dependencies (via a `-no-ui` lightweight build), it provides a stable, highly efficient, and native ARM64 TeX/LaTeX core engine toolchain with **fully automated, out-of-the-box deployment capabilities**.

> [!NOTE]
> This branch integrates the native Windows ARM64 compilation settings from the community PR [#1698](https://github.com/MiKTeX/miktex/pull/1698) (submitted by [@Grzzlwmpf](https://github.com/Grzzlwmpf)), which includes GMP platform flags, Asymptote header fixes, and CPU architecture manifests alignment.

---

## 🚀 Key Adaptations & Fixes

To achieve a native MSVC compile on Windows on ARM64 and guarantee automated package installation and system fonts working flawlessly, we implemented the following fixes in the source tree:

### 1. Graphics & Math Engine Compatibility
* **Pixman SIMD Fix**: Excluded x86-specific MMX/SSE assembly and source files when building on ARM64 to prevent compilation failures.
* **Cairo DirectWrite API Target Elevation**: Disabled `WITH_LEGACY_WINDOWS_SUPPORT` on ARM64 to elevate the target Windows version to Windows 10 (`0x0a00`). This ensures Cairo compiles successfully with modern DirectWrite interfaces like `IDWriteFontFace5`.
* **LuaJIT Exclusion**: Bypassed compiling the optional `luajit` subproject (its built-in assembly engine compiler `buildvm` lacks native support for MSVC ARM64 compilation) and fallback to standard Lua 5.3 to keep LuaTeX stable.

### 2. Parallel Build & Linker Fixes
* **WebApp Header Build Race Condition Fix**: Explicitly defined `OBJECT_DEPENDS` in CMake for WebApp compilation, ensuring auto-generated headers are fully generated before source files are built.
* **COM Proxy MIDL Compiling Fix**: Changed the MIDL compiler target environment `/env` from the default `amd64` to `arm64`. Swapped `/no_robust` with `/robust` on 64-bit platforms, entirely eliminating undefined symbol linker errors (LNK2001) for `ProxyFileInfo` and related proxy functions.

### 3. Packaged Bootstrapping & Fonts Support
* **Digitally Signed scripts.ini & Dummy Maps**: Packaged the official digitally signed `scripts.ini` file to pass MiKTeX's strict signature checks during dynamic package downloading. Included dummy maps (`dvips35.map`, `updmap.cfg`, etc.) in the package installation list to allow instant on-the-fly package downloads without manual FNDB initialization.
* **CJK System Fonts Mapping**: Configured `fonts.conf.in` template to explicitly include `WINDOWSFONTDIR`. XeTeX can now instantly index and use Windows built-in CJK fonts (e.g., `SimSun`, `SimHei`) out-of-the-box, resolving parallel database locking issues.

### 4. Automated Registry & Start Menu Setup (NSIS)
* **User-level PATH Registration**: Installs the binary path `\texmf\miktex\bin\x64` to the **current user's environment `PATH`** (User PATH) rather than the machine-wide PATH, cleaning it up upon uninstallation.
  * **Safe Append**: Utilizes a single-line PowerShell action to append the path, bypassing the legacy 1024-character NSIS PATH truncation limitation.
* **No-Privilege Installation (Per-User)**: The installer requests standard `user` execution level (**no UAC admin prompt required**), defaulting its directory to the user's local AppData folder (`%LOCALAPPDATA%\Programs\MiKTeX`).
* **Silent Link Creation**: Installs standard engine physical hard links (`xelatex.exe`, `pdflatex.exe`, `latex.exe`, etc.) silently on setup completion by running `miktex.exe links install`.
* **Start Menu Quick Launch**: Registers a `MiKTeX Command Prompt` shortcut in the user's Start Menu. Typing `MiKTeX` in the Windows Search Bar instantly reveals the command prompt shortcut, opening a fully initialized TeX console ready to run `xelatex`.

---

## 🛠️ Build & Packaging Instructions

### 1. Prerequisites
We use Microsoft Visual Studio 2022 to build for native ARM64.
First, fetch and install the required dependencies through `vcpkg` targeting the `arm64-windows` triplet:

```powershell
vcpkg install --triplet=arm64-windows
```

### 2. Compile Core Toolchain (no-ui)
Create a separate build folder and configure CMake to compile the core engines without UI dependencies (MFC/Qt):

```powershell
# 1. Create and navigate to the build directory
mkdir build-no-ui
cd build-no-ui

# 2. Configure CMake with vcpkg toolchain for ARM64 Release
cmake -G "Visual Studio 17 2022" -A ARM64 `
  -DCMAKE_TOOLCHAIN_FILE="<vcpkg_path>/scripts/buildsystems/vcpkg.cmake" `
  -DMIKTEX_UI_QT=OFF -DMIKTEX_UI_MFC=OFF -DWITH_UI=OFF -DWITH_MAN_PAGES=OFF ..

# 3. Generate prerequisite config files
cmake --build . --config Release --target gen-config-files

# 4. Compile in parallel
cmake --build . --config Release --parallel
```

### 3. Generate Installers (CPack)
Run the following CPack commands inside your `build-no-ui` folder to generate the package format of choice.

#### 📦 A. Setup Installer (NSIS `.exe`)
```powershell
cpack -G NSIS -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 B. Setup Installer (WiX `.msi`)
If packaging with WiX v4 compiler, specify your local WiX tool path in the command:
```powershell
cpack -G WIX `
  -D CPACK_WIX_VERSION=4 `
  -D CPACK_WIX_EXECUTABLE="<path_to_wix_executable>" `
  -D CPACK_PACKAGE_NAME="MiKTeX" `
  -D CPACK_NSIS_DISPLAY_NAME="MiKTeX" `
  -D CPACK_PACKAGE_INSTALL_DIRECTORY="MiKTeX" `
  -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 C. Portable Zip Package (`.zip`)
```powershell
cpack -G ZIP -D CPACK_PACKAGE_FILE_NAME="miktex-woa-portable"
```

---

## 📖 Installation & Usage

1. **One-Click Installation**: Double-click `MiKTeX-26.5-windows-arm64.exe` (or `.msi`) and follow the installer prompts. No admin prompt is required.
2. **Instant Use**: Once installed, **no manual configuration or path editing is needed**. Just open the Windows Start Menu, search for `MiKTeX`, and launch **`MiKTeX Command Prompt`**. You can compile your LaTeX project directly:
   ```powershell
   xelatex Thesis.tex
   ```
3. **Portable Mode**: If you want to keep your system Registry and `AppData` completely clean, unzip `miktex-woa-portable.zip` and directly use `texmf\miktex\bin\x64\xelatex.exe`. All downloaded styles/packages and configs will remain encapsulated inside the portable directory.
