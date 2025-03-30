# Real time on IBM PC-compatible systems ("x86")

Users switching between Windows and Linux sometimes
experience unexpected time jumps when rebooting.
This is commonly said to be caused by Windows saving local time to the RTC
and Linux saving UTC time to the RTC.

UEFI was supposed to fix that (see [this presentation](https://youtu.be/V2aq5M3Q76U?si=Q0bzhMK8Qe2O9tVS&t=861) by Matthew Garrett). UEFI includes [Time Services](https://uefi.org/specs/UEFI/2.10/08_Services_Runtime_Services.html#time-services)
as part of its Runtime Services that are available to the OS
after boot. This protocol exposes `GetTime()`/`SetTime()` methods
that save a UTC offset (= timezone information) along with the time.
Why is then the issue still present in 2025?

It seems that both Windows and Linux on x86 do *not* use the UEFI Time Services.
Rather, both seem to directly access the RTC CMOS via the x86 port I/O or MMIO.
This mechanism does not include timezone information, hence the issues with time jumps.

## What is happening where

### Linux

Linux has a `rtc-efi` driver. However, that driver is disabled on x86: [code](https://github.com/torvalds/linux/blob/93d52288679e29aaa44a6f12d5a02e8a90e742c5/drivers/rtc/Kconfig#L1191).
The reasoning in [the LKML submission](https://lore.kernel.org/all/1412348517.5410.13.camel@deneb.redhat.com/T/) is
that the UEFI implementations available at the time were buggy and were
able to crash the machine. Attempts to later allow the driver to be manually enabled
(see [email](https://lore.kernel.org/all/1493715444-79142-1-git-send-email-hehy1@lenovo.com/)) were rejected as unjustified.

Instead, it seems that RTC is accessed via the old I/O port mechanism
via the [rtc-cmos](https://github.com/torvalds/linux/blob/93d52288679e29aaa44a6f12d5a02e8a90e742c5/drivers/rtc/rtc-cmos.c) driver
and some [x86 glue](https://github.com/torvalds/linux/blob/93d52288679e29aaa44a6f12d5a02e8a90e742c5/arch/x86/kernel/rtc.c).

### Windows

According to my experiments with UEFI Shell, Windows on x86 seems to ignore the TimeZone field in UEFI.
* Initially I had everything in UTC with the UEFI timezone set to `UTC+00:00`.
* When I have `RealTimeIsUniversal=1` in Windows Registry (see below), Windows saves the UTC time to RTC and keeps the timezone field set to `UTC+00:00`.
* When I have `RealTimeIsUniversal=0`, Windows saves the local time to RTC. Unfortunately, it does not override the timezone field, keeping it set to `UTC+00:00`, which is incorrect where I live (`UTC+02:00` at the time).
* If I keep `RealTimeIsUniversal=1` and manually change the UEFI timezone in UEFI shell to `UTC+04:00`, Windows still shows correct time and also does not overwrite the timezone with a different one.

This leads me to believe that Windows also directly accesses the RTC, bypassing the UEFI Time Services.

This would make sense to me: if Linux was hitting crashes in some UEFI implementations,
it likely means that the code in them was not tested much. If the vendors were testing
only with Windows and Windows did not use Time Services, they would not find these bugs.

### UEFI

How is `GetTime()` actually implemented in UEFI? Fortunately, the core of UEFI is open-source
in the form of EDK2. In there, we can find the  following driver:
[PcatRealTimeClockRuntimeDxe/PcRtc.c](https://github.com/tianocore/edk2/blob/3907f8a0bab35c90a0d4ee0352a6fdec04b35bd2/PcAtChipsetPkg/PcatRealTimeClockRuntimeDxe/PcRtc.c).
This driver implements the Time Services on top of the x86 RTC:
* The raw time value seems to be written directly into the RTC via I/O ports (or MMIO).
* The timezone and DST information are stored in a `RTC` UEFI variable.

However, the driver seems to be used only on some platforms. My ThinkPad L13 Gen2 may have it,
but some AMD machines that I have access to seem not to have it (i.e. some other code is
implementing the Time Services).

## What to do about this as a user

In my opinion, the best way to work around this is to use the `RealTimeIsUniversal` Windows
registry key that is documented [on the Arch Wiki](https://wiki.archlinux.org/title/System_time).
This way, both Linux and Windows expect UTC time in the RTC.
