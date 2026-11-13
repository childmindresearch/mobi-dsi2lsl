# mobi-dsi2lsl

Streams data from a Wearable Sensing DSI headset to [Lab Streaming Layer (LSL)](https://labstreaminglayer.org).

This is a fork of [labstreaminglayer/App-WearableSensing](https://github.com/labstreaminglayer/App-WearableSensing) (branch `WearableSensing-master`) maintained by the Child Mind Institute. It builds two Windows programs:

- **`dsi2lsl.exe`**: the command-line streamer.
- **`dsi2lslGUI.exe`**: a GUI that runs `dsi2lsl.exe` in the background with the settings you enter.

> [!IMPORTANT]
> Windows only. Last updated November 2026.

## Changes from upstream

- **Accelerometer enabled.** Upstream never turned on the headset's 3D accelerometer, so the `ACCX`, `ACCY`, `ACCZ` and `ACCT` channels streamed zeros. It is now on at 30 Hz by default, the maximum documented by the DSI API.
- **Sampling rate setting.** The EEG sampling rate can be set. The GUI defaults to **600 Hz**.
- **New GUI controls:** a **Sampling Rate (Hz)** field (empty means 600) and an **On-board accelerometer enabled** checkbox (checked by default). Both are locked while streaming.
- **New command-line options:** `--sample-rate` and `--accel-rate`. See [Command-line options](#command-line-options).

> [!NOTE]
> Both features must be unlocked on your headset by Wearable Sensing: **Variable Acquisition Rate** for rates other than 300 Hz, and **Accelerometer**. If a feature is locked, the program prints a warning and keeps streaming. It stays at 300 Hz, or the accelerometer channels stay at zero. The console always prints what the headset actually reports:
> ```
> Sampling rate: requested 600 Hz, headset reports 600.0 Hz
> Accelerometer rate: requested 30 Hz, headset reports 30.0 Hz
> ```

The accelerometer updates at up to 30 Hz but is sent on every EEG sample, so each value repeats until the next reading arrives. Expect a "staircase" signal in the recording. The LSL metadata labels every channel as type `EEG` in microvolts, including the accelerometer channels.

## Requirements

| What | Where to get it |
|---|---|
| Visual Studio 2022 (Community is fine) with the **Desktop development with C++** workload | https://visualstudio.microsoft.com/downloads/ (includes CMake) |
| Qt 6 with an **MSVC 64-bit** component (e.g. `msvc2022_64`), or Qt 5.15.2 `msvc2019_64` | https://www.qt.io/download-dev |
| liblsl 1.16.2, **Win_amd64** build | https://github.com/sccn/liblsl/releases |
| DSI API v1.18.2 | https://wearablesensing.com/files/DSI-API_Current.zip |
| Git for Windows | https://git-scm.com/download/win |

Everything must be **64-bit**. Mixing in 32-bit pieces causes error `0xc000007b` at startup.

## Build

Run every command in **x64 Native Tools Command Prompt for VS 2022**, from the Start menu.

### 1. Check the tools

```bat
git --version
cmake --version
cl
```

`cl` should print `Microsoft (R) C/C++ Optimizing Compiler ... for x64`.

### 2. Clone

```bat
cd %USERPROFILE%\Documents
git clone https://github.com/childmindresearch/mobi-dsi2lsl.git
cd mobi-dsi2lsl
```

### 3. Add the DSI API

The API folder is not in the repository, so add it after every fresh clone. From the repository root:

```bat
curl -L -o DSI-API.zip https://wearablesensing.com/files/DSI-API_Current.zip
tar -xf DSI-API.zip
dir DSI_API_v1.18.2_04102023
```

The folder should contain `DSI.h`, `DSI_API_Loader.c` and `libDSI-Windows-x86_64.dll`. If the zip has a newer version with a different folder name, update the `DSI-API` path in [CMakeLists.txt](CMakeLists.txt).

### 4. Find Qt and liblsl

```bat
where /r C:\Qt Qt6Config.cmake
where /r C:\Qt Qt5Config.cmake
where /r %USERPROFILE% LSLConfig.cmake
where /r %USERPROFILE% lsl.dll
```

- **Qt:** use a result whose path contains `msvc` (e.g. `msvc2022_64`), not `mingw`. You need the **kit folder**: the part of the path before `\lib\cmake`, e.g. `C:\Qt\6.8.0\msvc2022_64`. Qt 6 is used if both are installed.
- **liblsl:** e.g. `...\liblsl-1.16.2-Win_amd64\lib\cmake\LSL` and `...\liblsl-1.16.2-Win_amd64\bin\lsl.dll`.
- **Nothing found for liblsl:** search `C:\` instead of `%USERPROFILE%`.

### 5. Configure and build

Replace the paths with what you found in step 4:

```bat
set LSLCFG=C:\path\to\liblsl-1.16.2-Win_amd64\lib\cmake\LSL
set QTKIT=C:\Qt\6.8.0\msvc2022_64
set LSLDLL=C:\path\to\liblsl-1.16.2-Win_amd64\bin\lsl.dll
set QTBIN=%QTKIT%\bin

cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DLSL_DIR="%LSLCFG%" -DCMAKE_PREFIX_PATH="%QTKIT%"
cmake --build build --config Release
```

This produces `build\Release\dsi2lsl.exe` and `build\Release\dsi2lslGUI.exe`.

### 6. Copy the DLLs

```bat
cd build\Release
"%QTBIN%\windeployqt.exe" dsi2lslGUI.exe
copy "%LSLDLL%" .
copy "..\..\DSI_API_v1.18.2_04102023\libDSI-Windows-x86_64.dll" libDSI.dll
```

The DSI DLL **must** be renamed to `libDSI.dll`. `build\Release` should now contain:

```
platforms\qwindows.dll
dsi2lsl.exe
dsi2lslGUI.exe
libDSI.dll
lsl.dll
Qt6Core.dll       (Qt5Core.dll with Qt 5)
Qt6Gui.dll
Qt6Widgets.dll
```

To run the programs on another computer, copy this whole folder.

## Usage

### GUI

Run `dsi2lslGUI.exe`. Fill in the fields, then click **Start**.

| Field | Meaning |
|---|---|
| Port | Headset COM port, e.g. `COM4`. Check Device Manager or the Bluetooth settings. |
| LSL Stream Name | Name of the LSL stream (default `WS-default`). |
| Montage | Comma-separated channel list, or `(use default)`. |
| Reference | Reference sensor, or `(use default)`. |
| Sampling Rate (Hz) | EEG rate. Empty means 600. |
| On-board accelerometer enabled | Turns the accelerometer on (30 Hz) or off. |
| Impedance / Reset | Impedance driver on/off, and reset. |

### Command line

```bat
dsi2lsl.exe --port=COM4 --sample-rate=600
```

Press Ctrl+C to stop. While it runs, type one of these commands and press Enter:

- `checkZOn`: turn the impedance driver on.
- `checkZOff`: turn the impedance driver off.
- `resetZ`: run an analog reset.

#### Command-line options

| Option | Default | Description |
|---|---|---|
| `--port` | `DSISerialPort` env var | Serial port, e.g. `COM4`. |
| `--montage` | headset default | Comma-separated channel list, e.g. `F3,F4,C3,C4,ACCX,ACCY,ACCZ`. |
| `--reference` | headset default | Reference sensor. |
| `--sample-rate` | headset's current rate (300) | EEG sampling rate in Hz. Needs Variable Acquisition Rate unlocked. |
| `--accel-rate` | `30` | Accelerometer rate in Hz; `0` turns it off. Needs Accelerometer unlocked. |
| `--lsl-stream-name` | `WS-default` | LSL stream name. |
| `--verbosity` | `2` | Amount of messages from the headset. |
| `--help` | | Show help. |

If the accelerometer channels are missing from the stream, add `ACCX,ACCY,ACCZ` to the montage.

## Troubleshooting

| Problem | Fix |
|---|---|
| `Could not find LSL` / `Could not find a package configuration file provided by "QT"` during configure | `LSLCFG` must be the folder that directly contains `LSLConfig.cmake`; `QTKIT` must be the Qt kit folder, e.g. `C:\Qt\6.8.0\msvc2022_64`. Delete the `build` folder before re-running configure. |
| `failed to load dynamic library "libDSI.dll"` | Copy and rename the DSI DLL into `build\Release` (step 6). |
| `lsl.dll was not found` | Copy `lsl.dll` into `build\Release`. |
| GUI says the `windows` platform plugin is missing | Run `windeployqt` again (step 6). |
| Error `0xc000007b` | A 32-bit DLL got mixed in. Use the `Win_amd64` liblsl, the `x86_64` DSI DLL and a 64-bit `msvc` Qt kit. |
| Rate reports 300 Hz / accelerometer warning | The feature is locked on the headset. Contact Wearable Sensing. |

## Alternative: MinGW build

`build-mingw.bat` builds both programs with MinGW-w64 and a MinGW build of Qt 5. Edit the `YOUR_PATH_...` placeholders in the script, then run `.\build-mingw.bat`. The output goes to `mingwbuild\`.

## Support

For issues with this fork, open an issue at https://github.com/childmindresearch/mobi-dsi2lsl. For issues with the original program or the DSI API, contact support@wearablesensing.com.
