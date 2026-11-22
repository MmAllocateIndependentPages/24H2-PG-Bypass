# 24H2 PG Bypass

PatchGuard bypass research driver for Windows 11 24H2 (build **10.0.26100**).

Shipped offsets are pulled from a specific 24H2 `ntoskrnl.exe` (see "Update to your winver" below for how to re-point them). The code paths themselves are not tied to a build, only the offset table is.

## Target

- OS: Windows 11 24H2
- Kernel build: 10.0.26100 (`ntoskrnl.exe` 10.0.26100.xxxx)
- Arch: x64 only

## How it works

The driver attacks every piece of PatchGuard's runtime from a different angle so even if one method misses (new context type, randomized dispatch, etc.) the others still strip it.

### Barricade (PTE NX flip + custom #PF handler)
- Flips the NX bit on PTEs that cover PatchGuard pages on a per-processor basis.
- Injects a custom interrupt handler straight into the IDT for execute-access faults.
- Installs a hook on `MmAccessFault` so PG's own validation paths come through our handler first.
- Because this operates at the page-fault layer, it catches PG regardless of how it got dispatched. APC method, system-thread method, and DPC-hook method all route through NX faults and all get neutralized here.

### Context 7 destroy (KiInitializePatchGuard global ctx)
- Patches `KiSwInterruptDispatch` with a `ret` so it stops dereferencing the global PG pointer (`MaxDataSize`), which would otherwise bugcheck once the global is cleared.
- Nulls out the `MaxDataSize` global pointer.
- Takes down the top-level PatchGuard context that `KiInitializePatchGuard` builds.

### Timer disable
- Walks all 512 PRCB timer table entries on every processor.
- Decrypts each `KTIMER`'s DPC pointer using the per-CPU timer key material.
- Identifies PatchGuard timers and unlinks / disables them.
- Covers both normal expiration and the "late-arming" DPCs PG queues for self-check.

### Misc mitigation
- Patches `KiMcaDeferredRecoveryService` to a plain `ret`. PG uses it as its call-stack scrubber before `KeBugCheckEx`, so cutting it short turns the sanity routines (`PspProcessDelete`, `KiInitializeUserApc` tail path) into no-ops as far as PG's post-check goes.
- Fixes PG-associated APCs still queued at init time.
- Takes down the PG worker system thread.

### VerifyPg (companion driver)
A small second driver you can load to actively try to trigger PatchGuard (write to a monitored kernel structure, patch a watched function, etc.) so you can confirm the bypass is holding without waiting for the normal randomized check window.

## Repo layout

```
24H2-PG-Bypass/
  pgscope.sln              VS solution
  pgscope/                 main bypass driver
    Entry.cpp              DriverEntry
    Global.hpp/.cpp        offset table + runtime variables
    Module/
      Barricade.cpp/.hpp   NX flip + custom #PF handler
      Context7.cpp/.hpp    MaxDataSize / KiSwInterruptDispatch
      Timer.cpp/.hpp       KTIMER table walk + PG timer kill
      Misc.cpp/.hpp        KiMcaDeferredRecoveryService + APC/thread fixups
    Util/                  arithmetic + memory helpers
    Log.hpp                debug print wrappers
  VerifyPg/                companion trigger driver for testing
  assets/                  misc
```

## Build

1. Install **Visual Studio 2022** with the **Desktop development with C++** workload (MSVC v143 toolset).
2. Install the **Windows Driver Kit** matching your SDK (10.0.26100).
3. Open `pgscope.sln`, pick **Release / x64**, build.
4. On the test machine:
   ```
   bcdedit /set testsigning on
   bcdedit /debug on            (optional, for WinDbg)
   shutdown /r /t 0
   ```
5. Install and start the driver (e.g. via `sc create` + `sc start`, or the OSR Driver Loader).

## Update to your winver

The only thing tied to a specific build is the offset table in [`pgscope/Global.hpp`](pgscope/Global.hpp) under `namespace Offsets`. Everything else is symbolic (`MmGetSystemRoutineAddress("KeBugCheckEx")` anchors the kernel base, every other symbol is `KernelBase + <offset>`).

To port to a different build:

1. Grab the `ntoskrnl.exe` from your target machine (`C:\Windows\System32\ntoskrnl.exe`).
2. Load it in IDA / Ghidra / BinaryNinja with symbols from Microsoft's symbol server.
3. For each entry in `Offsets`, resolve the RVA of the symbol:

   | Field | Symbol / meaning |
   |---|---|
   | `KeBugCheckExOff` | `KeBugCheckEx` RVA (anchors kernel base, must be exact) |
   | `KiWaitAlwaysOff` | `KiWaitAlways` |
   | `KiWaitNeverOff` | `KiWaitNever` |
   | `KeGetCurrentPrcbOff` | `KeGetCurrentPrcb` |
   | `CcBcbProfilerOff` | `CcBcbProfiler` |
   | `CcBcbProfiler2Off` | second `CcBcbProfiler` variant referenced by PG |
   | `MaxDataSizeOff` | global PG context pointer (xref from `KiInitializePatchGuard`) |
   | `KiSwInterruptDispatchOff` | `KiSwInterruptDispatch` |
   | `KiMcaDeferredRecoveryServiceOff` | `KiMcaDeferredRecoveryService` |
   | `MiVisibleStateOff` | `MiVisibleState` global |
   | `MmPteBaseOff` | load of `MmPteBase` inside kernel (self-reference base) |
   | `KeDelayExecutionTheadOff` | `KeDelayExecutionThread` |
   | `KeWaitForMultipleObjectsOff` | `KeWaitForMultipleObjects` |
   | `KeWaitForSingleObjectOff` | `KeWaitForSingleObject` |
   | `MmAccessFaultOff` | `MmAccessFault` |
   | `KiPageFaultOff` | `KiPageFault` |
   | `KiBalanceSetManagerDeferredRoutineOff` | `KiBalanceSetManagerDeferredRoutine` |
   | `KiBalanceSetManagerPeriodicDpcOff` | `KiBalanceSetManagerPeriodicDpc` |

4. Also re-verify the struct offsets in the same `Offsets` block:

   | Field | Where it comes from |
   |---|---|
   | `PrcbTimerTableOff` | `_KPRCB::TimerTable` offset (check `windbg: dt nt!_KPRCB`) |
   | `TimerTableEntryOff` | sizeof entry from `KTIMER_TABLE_ENTRY` |
   | `HalReservedOff` | `_KPRCB::HalReserved` offset |
   | `AcpiReservedOff` | `_KPRCB::AcpiReserved` offset |
   | `SystemVaTypeOff` | `_MI_VISIBLE_STATE::SystemVaType` offset |
   | `FaultingAddressOff` | displacement of faulting-address load inside `KiPageFault -> MmAccessFault` call. Disassemble `KiPageFault`, find the call to `MmAccessFault`, compute offset from function start + 5. |

5. Rebuild and reload.

If `InitializeRuntimeVariables` logs `Base mismatch! Please update offsets`, your `KeBugCheckExOff` is wrong, that's the first thing to re-check.

## Testing

Load `pgscope.sys` first, then optionally load `VerifyPg.sys` to actively poke at PG-watched structures. Watch for:
- No bugcheck (`CRITICAL_STRUCTURE_CORRUPTION`, `0x109`) after sustained write.
- Debug output prefix `[Pgscope]` in DbgView / WinDbg.

## License

MIT. See [LICENSE](LICENSE).
