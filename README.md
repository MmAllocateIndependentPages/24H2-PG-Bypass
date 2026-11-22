# pgscope

PatchGuard research driver for Windows 11 24H2 (build 10.0.26100).

## Target
- Windows 11 24H2
- Build 10.0.26100
- x64

## Components
- `pgscope/` kernel driver
- `VerifyPg/` verification harness
- `pgscope.sln` Visual Studio solution

## Build
Open `pgscope.sln` in Visual Studio with the WDK installed. Build x64. Enable test-signing on the target (`bcdedit /set testsigning on`).

## Notes
Research only. Disposable VM.
