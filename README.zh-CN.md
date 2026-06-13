# MiKTeX - Windows on ARM64 (WOA) 原生移植分支

[English Version](./README.md) | **中文说明**

本项目是官方 [MiKTeX](https://github.com/MiKTeX/miktex) 26.5 版本的 Fork 分支，专门针对 **Windows on ARM64 (WOA)** 架构进行了原生适配与编译优化。通过去除不必要的 Qt/MFC 图形界面依赖（进行 `-no-ui` 轻量化构建），实现了一套完全在 ARM64 架构下原生运行、稳定高效且具备**全自动开箱即用部署能力**的 TeX/LaTeX 核心编译链。

> [!NOTE]
> 本仓库已合并并致谢了来自上游社区 PR [#1698](https://github.com/MiKTeX/miktex/pull/1698)（由 [@Grzzlwmpf](https://github.com/Grzzlwmpf) 提交）的原生编译配置（包含 GMP 平台标志、Asymptote 头文件适配以及应用程序清单 CPU 架构对齐），确保与官方上游开发方向完全对齐。

---

## 🚀 相对于官方库的 Fork 适配改动

为了在 Windows on ARM64 环境下成功通过 MSVC 原生编译 MiKTeX 核心引擎，并保障自动依赖包下载和系统字体正常工作，本项目在官方源码及构建脚本中进行了以下关键性修复与适配：

### 1. 底层图形与数学引擎适配
* **Pixman SIMD 兼容修复**：排除了 x86 特有的 MMX/SSE 汇编源文件编译，避免 ARM64 MSVC 编译器报错。
* **Cairo 库 DirectWrite API 提升**：在 ARM64 平台下取消了老旧的 `WITH_LEGACY_WINDOWS_SUPPORT`（原将其限制在 Vista API），强制提升目标 Windows 平台版本至 Windows 10 (`0x0a00`)，以兼容 Cairo 对 `IDWriteFontFace5` 等现代 API 的调用。
* **LuaJIT 架构冲突规避**：剔除了 `luajit` 可选子模块的编译（其内置汇编虚拟机 `buildvm` 暂不支持 MSVC ARM64 交叉环境编译），转而直接使用原生轻量的 Lua 5.3 以确保 LuaTeX 稳定运作。

### 2. 编译树并发与链接故障修复
* **WebApp 并发编译头文件竞争修复**：为 WebApp 源文件和入口组件显式声明了 `OBJECT_DEPENDS` 规则，确保其依赖的自动生成头文件在编译前已经完整生成，根治了多线程并发编译时的文件丢失报错。
* **COM 代理组件 MIDL 编译修复**：将 MIDL 编译器的生成目标架构由默认的 `amd64` 修正为原生 `arm64`；为 64 位平台启用 `/robust` 编译参数替代与 x64 冲突的 `/no_robust`，彻底解决了 `ProxyFileInfo` 等代理存根符号在链接阶段报错未定义的 LNK2001 故障。

### 3. 打包自举资源固化与字体识别适配
* **自举 Map 与官方签名 scripts 固化**：在打包列表中强合入了带官方加密数字签名的 `scripts.ini`，避免由于硬链接程序对签名的校验失败而引发的致命崩溃；同时固化了自举所需的 `updmap.cfg` 以及 dummy map 文件（`dvips35.map` 等），无需任何手动环境初始化，即可秒级通畅联网自动下载所缺宏包。
* **系统中文字体自动识别**：在打包生成的 `fonts.conf.in` 字体配置模板中显式集成了 `WINDOWSFONTDIR`，使 xetex 等引擎能够直接精准识别和加载系统内置的中文字体（如宋体 `SimSun`、黑体 `SimHei` 等），彻底告别了中文字体检索异常导致的并发锁死问题。

### 4. 安装包一键部署与自注册（WiX v4 & NSIS）
* **系统 PATH 自动注册**：在安装包的构建配置中，直接动态注入了将实际可执行二进制路径 `\texmf\miktex\bin\x64` 注册到系统全局环境变量 `PATH` 的逻辑，并在卸载时安全剥离。
  * **NSIS (EXE)**: 创新性地在 PostInstall 执行段使用了单行无分号的 PowerShell 原生操作脚本，避开了 NSIS 传统变量在处理长 PATH（大于 1024 字节）时会导致 PATH 瘫痪性截断的漏洞。
  * **WiX (MSI)**: 采用 WiX v4 专业 XML 语法进行 Registry 的注入与维护。
* **常用引擎硬链接静默构建**：在打包配置中挂载了后置执行指令，在文件解压完成后，静默且具备完整管理员权限自动在后台执行 `miktex.exe --admin links install` 生成所有经典编译引擎（`xelatex.exe`, `pdflatex.exe`, `latex.exe` 等）的物理硬链接，真正做到**即装即用，彻底零手动配置**。

---

## 🛠️ 构建与编译打包指南

### 1. 编译前置准备
本项目采用 Microsoft Visual Studio 2022 进行原生 ARM64 编译。
在编译前，需要使用 `vcpkg` 配合指定的 arm64 三元组准备好所有依赖库（如 icu, libressl, expat, libpng, zlib, popt 等）：

```powershell
# 使用 vcpkg 安装原生 ARM64 依赖
vcpkg install --triplet=arm64-windows
```

### 2. CMake 编译核心工具链 (no-ui)
在项目根目录下，创建一个独立的构建文件夹，并通过 CMake 显式配置移去 Qt 与 MFC 的 UI 支持，只构建底层核心工具链：

```powershell
# 1. 创建并进入构建目录
mkdir build-no-ui
cd build-no-ui

# 2. CMake 配置（显式指定 VS2022 + ARM64 架构 + vcpkg 工具链 + 关闭 UI/Qt/MFC 编译）
cmake -G "Visual Studio 17 2022" -A ARM64 `
  -DCMAKE_TOOLCHAIN_FILE="C:/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake" `
  -DMIKTEX_UI_QT=OFF -DMIKTEX_UI_MFC=OFF -DWITH_UI=OFF -DWITH_MAN_PAGES=OFF ..

# 3. 预先构建配置文件目标
cmake --build . --config Release --target gen-config-files

# 4. 执行多核并行编译
cmake --build . --config Release --parallel
```

### 3. 生成发布包 (CPack)
在 `build-no-ui` 编译输出目录下，可根据您的需求通过 CPack 命令打包输出以下三种格式的包。

> [!IMPORTANT]
> 由于系统上只有新版 WiX v4 架构环境 (`wix.cmd`)，生成 MSI 包时，必须通过 `-D` 指明 WiX v4 的代理执行路径。

#### 📦 A. 生成统一安装包 (NSIS `.exe` 格式)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 B. 生成专业安装包 (WiX `.msi` 格式)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -G WIX `
  -D CPACK_WIX_VERSION=4 `
  -D CPACK_WIX_EXECUTABLE="C:\Users\Sunny\scoop\shims\wix.cmd" `
  -D CPACK_PACKAGE_NAME="MiKTeX" `
  -D CPACK_NSIS_DISPLAY_NAME="MiKTeX 26.5" `
  -D CPACK_PACKAGE_INSTALL_DIRECTORY="MiKTeX 26.5" `
  -D CPACK_PACKAGE_FILE_NAME="MiKTeX-26.5-windows-arm64"
```

#### 📦 C. 生成绿色免安装压缩包 (`.zip` 格式)
```powershell
C:\Users\Sunny\scoop\apps\cmake\current\bin\cpack.exe -G ZIP -D CPACK_PACKAGE_FILE_NAME="miktex-woa-portable"
```

---

## 📖 安装与使用说明

1. **一键安装**：双击下载好的 `MiKTeX-26.5-windows-arm64.msi`（或 `.exe`）安装程序，完成安装。
2. **免配置直接使用**：安装完成后，**无需运行任何初始化或环境变量配置脚本**。直接打开任意全新的终端（CMD/PowerShell）切到您的 LaTeX 项目目录，即可开始一键编译：
   ```powershell
   xelatex Thesis.tex
   ```
3. **便携模式 (Portable)**：如果您完全不想向系统注册表写入任何内容或修改 `AppData`，直接解压 `miktex-woa-portable.zip` 并使用解压目录下的 `texmf\miktex\bin\x64\xelatex.exe` 进行编译即可，它会将所有的缓存和配置文件完全保留在解压路径中，保持系统绝对纯净。
