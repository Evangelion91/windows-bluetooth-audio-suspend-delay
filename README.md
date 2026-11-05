# Windows 11 Bluetooth speaker clips the first sound after silence: testing BthA2dp SuspendDelay without a keep-alive

If the speaker stays connected but the first notification after 10–20 seconds is late or inaudible, the Windows A2DP stream may be suspended. On one Windows 11 Pro PC (OS build 26300, Microsoft BthA2dp.sys 10.0.26100.9711, Intel AX210 Bluetooth driver 24.70.0.4, Yandex Station Midi), delaying suspension improved short-gap playback. This is an undocumented, driver-wide setting, not a firmware modification or an always-on fix.

| Configuration | Idle gap | Observed WASAPI stream start call duration |
|---|---:|---:|
| Original, value absent | 1 s | 1 ms |
| Original, value absent | 18 s | 5001 ms; a subsequent trial failed while opening the endpoint |
| SuspendDelay = 120000 ms | 18 s | 1 ms in each of two trials |
| SuspendDelay = 120000 ms | 132 s | 1653 ms; the next start after 1 s took <1 ms |

Method: play a half-second tone through a shared WASAPI output stream, stop and close it, wait, then open and start a new stream. Time the start call separately from opening the stream. The longer-gap trial used a fresh process. Results are host-side timings, not microphone measurements or a controlled multi-device study. The speaker remained logically connected during the long-gap test. No continuous audio was used. With the change applied, one idle powercfg /requests snapshot showed no DISPLAY request; physical monitor sleep was not timed.

## Try and undo

1. Confirm your stereo endpoint uses the Microsoft Bluetooth A2DP driver. Test a short and a long gap before changing anything. This is not a fix for a speaker that powers off or disconnects entirely.
2. In an elevated command prompt, record the current value (including whether it is absent):
   ```bat
   reg query "HKLM\SYSTEM\CurrentControlSet\Services\BthA2dp\Parameters" /v SuspendDelay
   ```
3. To reproduce the measured configuration:
   ```bat
   reg add "HKLM\SYSTEM\CurrentControlSet\Services\BthA2dp\Parameters" /v SuspendDelay /t REG_DWORD /d 120000 /f
   ```
4. Restart the speaker's A2DP device in Device Manager, or reboot Windows. Repeat the tests below and above 120 seconds. Other Bluetooth stereo outputs on the PC can also be affected.
5. To undo, restore the recorded value and type. **Only if it was originally absent**, delete this value, then restart the same device or Windows:
   ```bat
   reg delete "HKLM\SYSTEM\CurrentControlSet\Services\BthA2dp\Parameters" /v SuspendDelay /f
   ```

## Why 214748 ms is the useful upper bound in this driver build

Using the installed driver and matching Microsoft public symbols: A2DP_Device::Initialize queries SuspendDelay as a DWORD under the driver's Parameters key, defaulting to zero if absent. A2DP_Device::TryToPauseStreaming checks the stored field. A2DP_Device::EnterNewStreamingState multiplies it by -10000 in a 32-bit register, then sign-extends the result to form a relative timer interval. 214748 ms converts to -2147480000; the next millisecond overflows. Do not interpret 0, a very large DWORD, or 0xffffffff as “never suspend”. The 214748 setting was applied and the output reappeared; behavior at that exact boundary was not timed. Other driver builds may differ.

Useful comparison: [Microsoft Bluetooth Classic Audio](https://learn.microsoft.com/windows-hardware/drivers/bluetooth/bluetooth-classic-audio) describes A2DP; [SoundKeeper](https://github.com/vrubleg/soundkeeper) takes the separate approach of keeping an audio stream active. A WinUSB policy also called SuspendDelay is unrelated.

Independent replication: report your BthA2dp.sys file version, OS build, adapter, speaker, before/after timings, and whether the first sound was physically audible after each gap. No serial numbers or Bluetooth addresses are needed.
