# Build instructions for Windows 64-bit

- [Prepare folder](#prepare-folder)
- [Install third party software](#install-third-party-software)
- [Initialize terminal](#initialize-terminal)
- [Clone source code and prepare libraries](#clone-source-code-and-prepare-libraries)
- [Build the project](#build-the-project)
- [Build with GitHub Actions](#build-with-github-actions)
- [Troubleshooting](#troubleshooting)

## Prepare folder

The build is done in **Visual Studio 2026** with **10.0.26100.0** SDK version.

Make sure you have these components installed with Visual Studio:

- MSVC v143 - VS 2022 C++ x64/x86 build tools (v14.44-17.14)
- C++ MFC v14.44 (x86 & x64)
- C++ ATL v14.44 (x86 & x64)
- Windows 11 SDK (10.0.26100.0)

Choose an empty folder for the future build, for example **D:\\TBuild**. It will be named ***BuildPath*** in the rest of this document. Create two folders there, ***BuildPath*\\ThirdParty** and ***BuildPath*\\Libraries**.

## Install third party software

* Download **Python 3.10** installer from [https://www.python.org/downloads/](https://www.python.org/downloads/) and install it with adding to PATH.
* Download **Git** installer from [https://git-scm.com/download/win](https://git-scm.com/download/win) and install it.

## Initialize terminal

Before preparing libraries and running build commands, initialize the Visual Studio environment for your target architecture.
The default modern toolset from Visual Studio 2026 (`v145`) does not support Windows 7, so for AyuGram Desktop you must use `-vcvars_ver=14.44` (`v144.4`, based on `v143` with Windows 7 support).

For Visual Studio installation:

    %comspec% /k "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvars64.bat" -vcvars_ver=14.44

For Visual Studio Build Tools installation:

    %comspec% /k "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\VC\Auxiliary\Build\vcvars64.bat" -vcvars_ver=14.44

Run both `Clone source code and prepare libraries` and `Build the project` sections in the terminal initialized with one of the commands above.

## Clone source code and prepare libraries

In the initialized terminal, go to ***BuildPath*** and run

    git clone --recursive https://github.com/AyuGram/AyuGramDesktop.git tdesktop
    tdesktop\Telegram\build\prepare\win.bat

## Build the project

Go to ***BuildPath*\\tdesktop\\Telegram** and run

    configure.bat x64 -D TDESKTOP_API_ID=2040 -D TDESKTOP_API_HASH=b18441a1ff607e10a989891a5462e627

* Open ***BuildPath*\\tdesktop\\out\\Telegram.slnx** in Visual Studio 2026
* Select Telegram project and press Build > Build Telegram (Debug and Release configurations)
* The result AyuGram.exe will be located in **D:\TBuild\tdesktop\out\Debug** (and **Release**)

## Build with GitHub Actions

The repository includes `.github/workflows/build.yml` for Windows x64 builds.
Push the workflow to the repository's default branch, enable GitHub Actions if
needed, then select **Actions > Build Windows x64 > Run workflow**. You can
select the branch and the `configuration` in the dispatch form. Choose
`Debug` (the default) or `Release`.

The workflow uses the API credentials shown above. To use your own credentials,
set the repository variable `TDESKTOP_API_ID` and the repository secret
`TDESKTOP_API_HASH` under **Settings > Secrets and variables > Actions**.

After a successful run, download `AyuGram-windows-x64-Debug` or
`AyuGram-windows-x64-Release` from the run's **Artifacts** section.
Debug builds also upload full PDB symbols separately as
`AyuGram-windows-x64-Debug-symbols`, which can be used without the runner's
object files and libraries. Dependencies are cached separately for each
configuration; the first run of each configuration prepares them from source.

The application build runs one MSBuild project at a time with up to two MSVC
compilation workers to reduce precompiled-header memory usage.

## Troubleshooting

### Error building libvpx

Downgrade NASM in msys64 to 3.01.

### PDB API call failed

From the `tdesktop` folder, run the following command in PowerShell:

```powershell
@'
diff --git a/options_win.cmake b/options_win.cmake
index dd86342..857d5b7 100644
--- a/options_win.cmake
+++ b/options_win.cmake
@@ -34,2 +34,3 @@ if (MSVC)
         /MP     # Enable multi process build.
+        /FS
         /EHsc   # Catch C++ exceptions only, extern C functions never throw a C++ exception.
diff --git a/variables.cmake b/variables.cmake
index a1e77b7..9f816cb 100644
--- a/variables.cmake
+++ b/variables.cmake
@@ -28,4 +28,4 @@ set(CMAKE_CXX_SCAN_FOR_MODULES OFF CACHE BOOL "")
 set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT
-    "$<$<CONFIG:Debug>:ProgramDatabase>$<$<NOT:$<CONFIG:Debug>>:Embedded>"
-    CACHE STRING "")
+    "$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>$<$<CONFIG:Release,MinSizeRel>:Embedded>"
+    CACHE STRING "" FORCE)
 set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "")
'@ | git -C cmake apply -
```
