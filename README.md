# MiKTeX - Native Port for Windows on ARM64 (WOA)

**English** | [中文说明](./README.zh-CN.md)

This project is a fork of the official [MiKTeX](https://github.com/MiKTeX/miktex) (v26.5 / `build-26.5` branch) tailored specifically for **Windows on ARM64 (WOA)**. By removing heavy Qt/MFC graphical interface dependencies (via a `-no-ui` lightweight build), it provides a stable, highly efficient, and native ARM64 TeX/LaTeX core engine toolchain with **fully automated, out-of-the-box deployment capabilities**.

> [!NOTE]
> This repository integrates and upstream-credits the native compilation settings from the community PR [#1698](https://github.com/MiKTeX/miktex/pull/1698) submitted by [@Grzzlwmpf](https://github.com/Grzzlwmpf) (such as GMP platform flags, Asymptote Windows.h headers, and Manifest CPU architecture alignment), ensuring full technical alignment with the official upstream development.

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
* **Digitally Signed scripts.ini & Dummy Maps**: Packaged the official digitally signed `scripts.ini` file to pass MiKTeX's strict signature checks during `links install` setup. Included dummy maps (`dvips35.map`, `updmap.cfg`, etc.) in the package installation list to allow instant on-the-fly package downloads without any manual FNDB initialization.
* **CJK System Fonts Mapping**: Configured `fonts.conf.in` template to explicitly include `WINDOWSFONTDIR` and `C:/Windows/Fonts`. XeTeX can now instantly index and use Windows built-in CJK fonts (e.g., `SimSun`, `SimHei`) out-of-the-box, resolving parallel database locking issues.

### 4. Automated Installer Registry & Link Setup (WiX v4 & NSIS)
* **Automatic PATH Registration**: Built-in installer scripts now dynamically calculate and append the nested binary path `\texmf\miktex\bin\x64` to the system-wide `PATH` environment variable, cleaning it up cleanly upon uninstallation.
  * **NSIS (EXE)**: Utilizes a custom semicolon-free single-line PowerShell script to append/remove the path, avoiding the legacy NSIS limit (PATH truncation if it exceeds 1024 bytes).
  * **WiX (MSI)**: Integrates modern WiX v4 Registry/Environment XML components.
* **Engine Links Creation**: Triggers a post-installation custom action that runs `miktex.exe --admin links install` silently with elevated privileges, generating physical hard links for standard engines (`xelatex.exe`, `pdflatex.exe`, `latex.exe`, etc.) automatically. **Users can write LaTeX files immediately after a double-click install.**

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
  -DCMAKE_TOOLCHAIN_FILE="C:/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake" `
  -DMIKTEX_UI_QT=OFF -DMIKTEX_UI_MFC=OFF -DWITH_UI=OFF -DWITH_MAN_PAGES=OFF ..

# 3. Generate prerequisite config files
cmake --build . --config Release --target gen-config-files

# 4. Compile in parallel
cmake --build . --config Release --parallel
```

### 3. Generate Installers (CPack)
Run the following commands inside your `build-no-ui` folder to generate the installers.

> [!IMPORTANT]
> Since only WiX v4 is installed on your machine (`wix.cmd`), we must supply additional variables to target the WiX v4 compiler bridge during MSI packaging.

#### 📦 A. Unified Installer (NSIS `.exe`)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 B. Professional Installer (WiX `.msi`)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -G WIX `
  -D CPACK_WIX_VERSION=4 `
  -D CPACK_WIX_EXECUTABLE="C:\Users\Sunny\scoop\shims\wix.cmd" `
  -D CPACK_PACKAGE_NAME="MiKTeX" `
  -D CPACK_NSIS_DISPLAY_NAME="MiKTeX 26.5" `
  -D CPACK_PACKAGE_INSTALL_DIRECTORY="MiKTeX 26.5" `
  -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 C. Green Portable Package (`.zip`)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -G ZIP -D CPACK_PACKAGE_FILE_NAME="miktex-woa-portable"
```

---

## 📖 Installation & Usage

1. **One-Click Installation**: Double-click `MiKTeX-26.5-windows-arm64.msi` (or `.exe`) and follow the installer prompts.
2. **Instant Use**: Once installed, **no manual configuration or path editing is needed**. Just open a new Command Prompt or PowerShell, navigate to your LaTeX directory, and run:
   ```powershell
   xelatex Thesis.tex
   ```
3. **Portable Mode**: If you want to keep your system Registry and `AppData` completely clean, unzip `miktex-woa-portable.zip` and directly use `texmf\miktex\bin\x64\xelatex.exe`. All downloaded styles/packages and configs will remain encapsulated inside the portable directory.
