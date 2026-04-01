## Build guide Windows

### Repository structure

This repo contains two git submodules side by side:

```
dev/
  json-scada/           # JSON-SCADA platform
  S7CommPlusDriver/     # Siemens S7 Communication Plus driver
```

The `S7CommPlusClient` (in `json-scada/src/S7CommPlusClient`) references the driver via a relative project path (`../../../S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusDriver.csproj`), so both submodules must be present.

### Prerequisites

- **Visual Studio 2022** (or Build Tools) with C++ and .NET workloads
- **.NET SDK 8.0**
- **Go 1.22+**
- **Node.js 20+** (must be in system PATH)
- **CMake** - for the libiec61850 native build
- **NSIS** (Nullsoft Scriptable Install System) - for building the installer
  - Add to PATH: `C:\Program Files (x86)\NSIS`
  - Install the **AccessControl** plugin: download from https://nsis.sourceforge.io/AccessControl_plug-in, then copy `Plugins\i386-unicode\AccessControl.dll` to `C:\Program Files (x86)\NSIS\Plugins\x86-unicode\` (note the folder name difference)

### 1. Clone with submodules

```cmd
git clone --recurse-submodules <repo-url>
```

Or if already cloned:

```cmd
git submodule update --init --recursive
```

### 2. Populate bundled runtimes and third-party binaries

The NSIS installer script packages many pre-downloaded runtimes and tools that are not stored in git. The easiest way to obtain these is to download and install the latest JSON-SCADA Windows release with the precompiled setup.exe. Next copy the binaries located in `c:\json-scada\platform-windows` to your checkout.

**Missing standalone binaries** (place in `json-scada\platform-windows\`):

| File | Source |
|------|--------|
| `nssm.exe` | https://nssm.cc/download |
| `ffmpeg.exe` | https://ffmpeg.org/download.html (win64 build) |
| `sounder.exe` | JSON-SCADA release assets |
| `OpcWatch.exe` | JSON-SCADA release assets |
| `vc_redist.x64.exe` | Microsoft Visual C++ Redistributable |
| `aspnetcore-runtime-8.0.23-win-x64.exe` | https://dotnet.microsoft.com/download |
| `dotnet-runtime-8.0.23-win-x64.exe` | https://dotnet.microsoft.com/download |
| `dotnet-runtime-10.0.2-win-x64.exe` | https://dotnet.microsoft.com/download |

**Missing runtime directories** (place in `json-scada\platform-windows\`):

| Directory | Contents |
|-----------|----------|
| `nodejs-runtime/` | Bundled Node.js distribution |
| `mongodb-runtime/` | MongoDB server binaries |
| `mongodb-compass-runtime/` | MongoDB Compass GUI |
| `postgresql-runtime/` | PostgreSQL server binaries |
| `grafana-runtime/` | Grafana server |
| `metabase-runtime/` | Metabase JAR (`metabase.jar`) |
| `jdk-runtime/` | Java JDK (for Metabase) |
| `nginx_php-runtime/` | Nginx + PHP binaries |
| `browser-runtime/` | Bundled Chromium |
| `browser-data/` | Chromium user data template |
| `telegraf-runtime/` | Telegraf binary + config files |
| `inkscape-runtime/` | Inkscape + SCADA extension |
| `ua-edge-translator-runtime/` | OPC UA Edge Translator |

### 3. Create a short-path junction

NSIS does not support long paths. The deeply nested files in some runtime directories exceed Windows' 260-character path limit when combined with a deep checkout path. Create a directory junction so NSIS sees a short base path:

```cmd
mklink /J C:\jsbuild "c:\path\to\your\checkout\json-scada"
```

All subsequent build and packaging commands must be run through `C:\jsbuild`, not the real path.

### 4. Configure build.bat

`JSPATH` in `json-scada\platform-windows\build.bat` is pre-set to `C:\jsbuild`. If your junction path differs, update it:

```bat
set JSPATH=C:\jsbuild
```

The script uses system-installed `npm`/`npx` from PATH (not the bundled `nodejs-runtime` directory which is only present in release builds).

### 5. Generate SSL certificate

`nginx.key` is excluded from git (`*.key` is in `.gitignore`). Generate a self-signed cert once — the same cert/key can be reused across all installations:

```cmd
cd C:\jsbuild\conf-templates
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout nginx.key -out nginx.crt -subj "/CN=localhost"
```

### 6. Build all binaries

Open a **Developer Command Prompt for VS 2022** (required for `msbuild` and .NET tooling) and run:

```cmd
C:\jsbuild\platform-windows\build.bat
```

This compiles all components:
- **.NET projects**: IEC 61850 client, IEC 101/104 clients and servers, DNP3 client, OPC-DA/UA clients, PLCTags client, **S7CommPlusClient** (+ S7CommPlusDriver), logrotate
- **Go projects**: calculations, i104m, plc4x-client
- **Node.js projects**: all frontend and backend Node modules including AdminUI (with custom Vue components)

All compiled binaries are output to `json-scada\bin\`.

### 7. Build the runtime pack (once, or when runtimes change)

The installer is split into a main installer (application code, rebuilt each release) and a separate runtime pack (large third-party runtimes, rarely changes). The runtime pack must be built once before the first installer build, and again only when runtimes are updated.

Requires **7-Zip** installed at `C:\Program Files\7-Zip\`.

```cmd
C:\jsbuild\platform-windows\build-runtime-pack.bat
```

Output: `json-scada\platform-windows\installer-release\json-scada-runtimes.7z`

### 8. Build the main installer

After all binaries are built:

```cmd
cd C:\jsbuild\platform-windows
makensis json-scada.nsi
```

Output: `json-scada\platform-windows\installer-release\json-scada_setup_v.0.61.exe`

### End-user installation

The end user receives two files and must apply them in order:

1. Extract `json-scada-runtimes.7z` to `C:\json-scada\` using 7-Zip
2. Run `json-scada_setup_v.0.61.exe`

The NSIS installer will abort with a clear error message if step 1 was skipped.

### Additions included in the build

- **S7CommPlusClient** (`json-scada/src/S7CommPlusClient`) - compiled by `build.bat`, binary packaged via the `bin\*.*` wildcard in the NSIS script
- **S7CommPlusDriver** (`S7CommPlusDriver/`) - compiled as a dependency of S7CommPlusClient, output DLLs (including OpenSSL) land in `bin\`
- **Custom Vue components** (S7PlusAlarmsViewerPage, DatablockBrowserPage, TagTreeBrowserPage) - built as part of AdminUI's `npm run build`, packaged via the `AdminUI\dist\*.*` entry in the NSIS script
