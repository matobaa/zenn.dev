---
title: "Mac Mini M4から CH32V203でLチカする (PlatformIO)"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wch", "ch32v", "cv32v203", "macos" ]
published: false
---

## tl;dr

新しく手に入れたM4MacでLチカしたかった。

## やったこと

1. homebrew 4.4.19 https://brew.sh/ja/
1. vscodeを導入
1. VSCode上にPlatformIOを導入
1. PlayformIO上にWCH32V203ボード設定を追加
1. 新規プロジェクトを作ってビルド……失敗。main がない
1. 新規ファイルを作って int main(void) {} を追加、ビルド……成功
1. アップロード……失敗。brew install libusb
1. アップロード……失敗。brew install hidapi
1. PlatformIOのHome>Projects>作成したプロジェクト>Configure>optionsのupload protocol に isp と記入してsave
1. アップロード……失敗。Error: Verify failed, mismatch
https://github.com/ch32-rs/wchisp/issues/29#issuecomment-1625646701 ここに答えがあった
wchisp config unprotect
before trying to flash your board.

以下あしあと

```
----
 *  Executing task: platformio run --target upload 

Processing genericCH32V203C8T6 (platform: ch32v; board: genericCH32V203C8T6; framework: noneos-sdk)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ch32v/genericCH32V203C8T6.html
PLATFORM: WCH CH32V (1.1.0+sha.8af7925) > Generic CH32V203C8T6
HARDWARE: CH32V203C8T6 144MHz, 20KB RAM, 64KB Flash
DEBUG: Current (wch-link) On-board (wch-link) External (minichlink)
PACKAGES: 
 - framework-wch-noneos-sdk @ 2.30000.0+sha.ce988d6 
 - tool-minichlink @ 0.1.0+sha.af02ba5 
 - tool-openocd-riscv-wch @ 2.1100.240729 (11.0) 
 - tool-wchisp @ 0.23.240914 
 - tool-wlink @ 0.23.241116+sha.0c802d4 
 - toolchain-riscv @ 1.80200.190731+sha.99cb62f
LDF: Library Dependency Finder -> https://bit.ly/configure-pio-ldf
LDF Modes: Finder ~ chain, Compatibility ~ soft
Found 0 compatible libraries
Scanning dependencies...
No dependencies
Building in release mode
Checking size .pio/build/genericCH32V203C8T6/firmware.elf
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  10.0% (used 2048 bytes from 20480 bytes)
Flash: [          ]   1.1% (used 736 bytes from 65536 bytes)
Configuring upload protocol...
AVAILABLE: isp, minichlink, wch-link, wlink
CURRENT: upload_protocol = wch-link
Uploading .pio/build/genericCH32V203C8T6/firmware.elf
dyld[33870]: Library not loaded: /opt/homebrew/opt/libusb/lib/libusb-1.0.0.dylib
  Referenced from: <CBDCC2EE-BD6E-426C-84A1-068DE383EE8A> /Users/matobaa/.platformio/packages/tool-openocd-riscv-wch/bin/openocd
  Reason: tried: '/opt/homebrew/opt/libusb/lib/libusb-1.0.0.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/opt/libusb/lib/libusb-1.0.0.dylib' (no such file), '/opt/homebrew/opt/libusb/lib/libusb-1.0.0.dylib' (no such file)
*** [upload] Error -6
========================================================================================= [FAILED] Took 0.46 seconds =========================================================================================

 *  The terminal process "platformio 'run', '--target', 'upload'" terminated with exit code: 1. 
 *  Terminal will be reused by tasks, press any key to close it. 

 *  Executing task: platformio run --target upload 

Processing genericCH32V203C8T6 (platform: ch32v; board: genericCH32V203C8T6; framework: noneos-sdk)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ch32v/genericCH32V203C8T6.html
PLATFORM: WCH CH32V (1.1.0+sha.8af7925) > Generic CH32V203C8T6
HARDWARE: CH32V203C8T6 144MHz, 20KB RAM, 64KB Flash
DEBUG: Current (wch-link) On-board (wch-link) External (minichlink)
PACKAGES: 
 - framework-wch-noneos-sdk @ 2.30000.0+sha.ce988d6 
 - tool-minichlink @ 0.1.0+sha.af02ba5 
 - tool-openocd-riscv-wch @ 2.1100.240729 (11.0) 
 - tool-wchisp @ 0.23.240914 
 - tool-wlink @ 0.23.241116+sha.0c802d4 
 - toolchain-riscv @ 1.80200.190731+sha.99cb62f
LDF: Library Dependency Finder -> https://bit.ly/configure-pio-ldf
LDF Modes: Finder ~ chain, Compatibility ~ soft
Found 0 compatible libraries
Scanning dependencies...
No dependencies
Building in release mode
Checking size .pio/build/genericCH32V203C8T6/firmware.elf
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  10.0% (used 2048 bytes from 20480 bytes)
Flash: [          ]   1.1% (used 736 bytes from 65536 bytes)
Configuring upload protocol...
AVAILABLE: isp, minichlink, wch-link, wlink
CURRENT: upload_protocol = wch-link
Uploading .pio/build/genericCH32V203C8T6/firmware.elf
dyld[35364]: Library not loaded: /opt/homebrew/opt/hidapi/lib/libhidapi.0.dylib
  Referenced from: <CBDCC2EE-BD6E-426C-84A1-068DE383EE8A> /Users/matobaa/.platformio/packages/tool-openocd-riscv-wch/bin/openocd
  Reason: tried: '/opt/homebrew/opt/hidapi/lib/libhidapi.0.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/opt/hidapi/lib/libhidapi.0.dylib' (no such file), '/opt/homebrew/opt/hidapi/lib/libhidapi.0.dylib' (no such file)
*** [upload] Error -6
========================================================================================= [FAILED] Took 0.37 seconds =========================================================================================

 *  The terminal process "platformio 'run', '--target', 'upload'" terminated with exit code: 1. 
 *  Terminal will be reused by tasks, press any key to close it. 

 *  Executing task: platformio run --target upload 

Processing genericCH32V203C8T6 (platform: ch32v; board: genericCH32V203C8T6; framework: noneos-sdk)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ch32v/genericCH32V203C8T6.html
PLATFORM: WCH CH32V (1.1.0+sha.8af7925) > Generic CH32V203C8T6
HARDWARE: CH32V203C8T6 144MHz, 20KB RAM, 64KB Flash
DEBUG: Current (wch-link) On-board (wch-link) External (minichlink)
PACKAGES: 
 - framework-wch-noneos-sdk @ 2.30000.0+sha.ce988d6 
 - tool-minichlink @ 0.1.0+sha.af02ba5 
 - tool-openocd-riscv-wch @ 2.1100.240729 (11.0) 
 - tool-wchisp @ 0.23.240914 
 - tool-wlink @ 0.23.241116+sha.0c802d4 
 - toolchain-riscv @ 1.80200.190731+sha.99cb62f
LDF: Library Dependency Finder -> https://bit.ly/configure-pio-ldf
LDF Modes: Finder ~ chain, Compatibility ~ soft
Found 0 compatible libraries
Scanning dependencies...
No dependencies
Building in release mode
Checking size .pio/build/genericCH32V203C8T6/firmware.elf
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  10.0% (used 2048 bytes from 20480 bytes)
Flash: [          ]   1.1% (used 736 bytes from 65536 bytes)
Configuring upload protocol...
AVAILABLE: isp, minichlink, wch-link, wlink
CURRENT: upload_protocol = wch-link
Uploading .pio/build/genericCH32V203C8T6/firmware.elf
Open On-Chip Debugger 0.11.0+dev-02415-gfad123a16-dirty (2024-08-09-10:49)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
debug_level: 1

Warn : Transport "sdi" was already selected
Ready for Remote Connections
Error: open failed

*** [upload] Error 1
========================================================================================= [FAILED] Took 0.43 seconds =========================================================================================

 *  The terminal process "platformio 'run', '--target', 'upload'" terminated with exit code: 1. 
 *  Terminal will be reused by tasks, press any key to close it. 

 *  Executing task: platformio run --target upload 

Processing genericCH32V203C8T6 (platform: ch32v; board: genericCH32V203C8T6; framework: noneos-sdk)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ch32v/genericCH32V203C8T6.html
PLATFORM: WCH CH32V (1.1.0+sha.8af7925) > Generic CH32V203C8T6
HARDWARE: CH32V203C8T6 144MHz, 20KB RAM, 64KB Flash
DEBUG: Current (wch-link) On-board (wch-link) External (minichlink)
PACKAGES: 
 - framework-wch-noneos-sdk @ 2.30000.0+sha.ce988d6 
 - tool-minichlink @ 0.1.0+sha.af02ba5 
 - tool-openocd-riscv-wch @ 2.1100.240729 (11.0) 
 - tool-wchisp @ 0.23.240914 
 - tool-wlink @ 0.23.241116+sha.0c802d4 
 - toolchain-riscv @ 1.80200.190731+sha.99cb62f
LDF: Library Dependency Finder -> https://bit.ly/configure-pio-ldf
LDF Modes: Finder ~ chain, Compatibility ~ soft
Found 1 compatible libraries
Scanning dependencies...
No dependencies
Building in release mode
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSCore/core_riscv.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSStartup/startup_ch32v20x_D6.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSSSystem/system_ch32v20x.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSDebug/debug.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkInitFini/cpp_support.o
Compiling .pio/build/genericCH32V203C8T6/src/main.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_adc.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_bkp.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_can.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_crc.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_dbgmcu.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_dma.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_exti.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_flash.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_gpio.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_i2c.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_iwdg.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_misc.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_opa.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_pwr.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_rcc.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_rtc.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_spi.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_tim.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_usart.o
Compiling .pio/build/genericCH32V203C8T6/FrameworkNoneOSVariant/ch32v20x_wwdg.o
Archiving .pio/build/genericCH32V203C8T6/libFrameworkNoneOSVariant.a
Indexing .pio/build/genericCH32V203C8T6/libFrameworkNoneOSVariant.a
Linking .pio/build/genericCH32V203C8T6/firmware.elf
Checking size .pio/build/genericCH32V203C8T6/firmware.elf
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  10.0% (used 2048 bytes from 20480 bytes)
Flash: [          ]   1.1% (used 736 bytes from 65536 bytes)
Configuring upload protocol...
AVAILABLE: isp, minichlink, wch-link, wlink
CURRENT: upload_protocol = isp
Uploading .pio/build/genericCH32V203C8T6/firmware.elf
05:46:52 [INFO] Chip: CH32V203C8T6[0x3119] (Code Flash: 64KiB)
05:46:52 [INFO] Chip UID: CD-AB-D5-F3-10-BC-B2-5B
05:46:52 [INFO] BTVER(bootloader ver): 02.70
05:46:52 [INFO] Code Flash protected: true
05:46:52 [INFO] Current config registers: ff003fc000ff00ffffffffff00020700cdabd5f310bcb25b
RDPR_USER: 0xC03F00FF
  [7:0]   RDPR 0xFF (0b11111111)
    `- Protected
  [16:16] IWDG_SW 0x1 (0b1)
    `- IWDG enabled by the software, and disabled by hardware
  [17:17] STOP_RST 0x1 (0b1)
    `- Disable
  [18:18] STANDBY_RST 0x1 (0b1)
    `- Disable, entering standby-mode without RST
  [23:22] SRAM_CODE_MODE 0x0 (0b0)
    `- CODE-192KB + RAM-128KB / CODE-128KB + RAM-64KB depending on the chip
DATA: 0xFF00FF00
  [7:0]   DATA0 0x0 (0b0)
  [23:16] DATA1 0x0 (0b0)
WRP: 0xFFFFFFFF
  `- Unprotected
05:46:52 [INFO] Read .pio/build/genericCH32V203C8T6/firmware.elf as ELF format
05:46:52 [INFO] Found loadable segment, physical address: 0x00000000, virtual address: 0x00000000, flags: 0x5
05:46:52 [INFO] Section names: [".init", ".vector", ".text"]
05:46:52 [INFO] Firmware size: 1024
05:46:52 [INFO] Erasing...
05:46:52 [WARN] erase_code: set min number of erased sectors to 8
05:46:52 [INFO] Erased 8 code flash sectors
05:46:53 [INFO] Erase done
05:46:53 [INFO] Writing to code flash...
05:46:53 [INFO] Code flash 1024 bytes written
05:46:54 [INFO] Verifying...
Error: Verify failed, mismatch
*** [upload] Error 1
========================================================================================= [FAILED] Took 2.74 seconds =========================================================================================

 *  The terminal process "platformio 'run', '--target', 'upload'" terminated with exit code: 1. 
 *  Terminal will be reused by tasks, press any key to close it. 

  *  Executing task: platformio run --target upload --environment genericCH32V203C8T6 

Processing genericCH32V203C8T6 (platform: ch32v; board: genericCH32V203C8T6; framework: noneos-sdk)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ch32v/genericCH32V203C8T6.html
PLATFORM: WCH CH32V (1.1.0+sha.8af7925) > Generic CH32V203C8T6
HARDWARE: CH32V203C8T6 144MHz, 20KB RAM, 64KB Flash
DEBUG: Current (wch-link) On-board (wch-link) External (minichlink)
PACKAGES: 
 - framework-wch-noneos-sdk @ 2.30000.0+sha.ce988d6 
 - tool-minichlink @ 0.1.0+sha.af02ba5 
 - tool-openocd-riscv-wch @ 2.1100.240729 (11.0) 
 - tool-wchisp @ 0.23.240914 
 - tool-wlink @ 0.23.241116+sha.0c802d4 
 - toolchain-riscv @ 1.80200.190731+sha.99cb62f
LDF: Library Dependency Finder -> https://bit.ly/configure-pio-ldf
LDF Modes: Finder ~ chain, Compatibility ~ soft
Found 1 compatible libraries
Scanning dependencies...
No dependencies
Building in release mode
Checking size .pio/build/genericCH32V203C8T6/firmware.elf
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  10.8% (used 2204 bytes from 20480 bytes)
Flash: [=         ]  11.1% (used 7268 bytes from 65536 bytes)
Configuring upload protocol...
AVAILABLE: isp, minichlink, wch-link, wlink
CURRENT: upload_protocol = isp
Uploading .pio/build/genericCH32V203C8T6/firmware.elf
08:58:04 [INFO] Chip: CH32V203C8T6[0x3119] (Code Flash: 64KiB)
08:58:04 [INFO] Chip UID: CD-AB-D5-F3-10-BC-B2-5B
08:58:04 [INFO] BTVER(bootloader ver): 02.70
08:58:04 [INFO] Code Flash protected: false
08:58:04 [INFO] Current config registers: a55a3fc000ff00ffffffffff00020700cdabd5f310bcb25b
RDPR_USER: 0xC03F5AA5
  [7:0]   RDPR 0xA5 (0b10100101)
    `- Unprotected
  [16:16] IWDG_SW 0x1 (0b1)
    `- IWDG enabled by the software, and disabled by hardware
  [17:17] STOP_RST 0x1 (0b1)
    `- Disable
  [18:18] STANDBY_RST 0x1 (0b1)
    `- Disable, entering standby-mode without RST
  [23:22] SRAM_CODE_MODE 0x0 (0b0)
    `- CODE-192KB + RAM-128KB / CODE-128KB + RAM-64KB depending on the chip
DATA: 0xFF00FF00
  [7:0]   DATA0 0x0 (0b0)
  [23:16] DATA1 0x0 (0b0)
WRP: 0xFFFFFFFF
  `- Unprotected
08:58:04 [INFO] Read .pio/build/genericCH32V203C8T6/firmware.elf as ELF format
08:58:04 [INFO] Found loadable segment, physical address: 0x00000000, virtual address: 0x00000000, flags: 0x5
08:58:04 [INFO] Section names: [".init", ".vector", ".text"]
08:58:04 [INFO] Found loadable segment, physical address: 0x00001bdc, virtual address: 0x20000000, flags: 0x6
08:58:04 [INFO] Section names: [".data"]
08:58:04 [INFO] Firmware size: 8192
08:58:04 [INFO] Erasing...
08:58:04 [INFO] Erased 9 code flash sectors
08:58:05 [INFO] Erase done
08:58:05 [INFO] Writing to code flash...
08:58:05 [INFO] Code flash 8192 bytes written
08:58:05 [INFO] Verifying...
08:58:05 [INFO] Verify OK
08:58:05 [INFO] Now reset device and skip any communication errors
08:58:05 [INFO] Device reset
======================================================================================== [SUCCESS] Took 1.89 seconds ========================================================================================
 *  Terminal will be reused by tasks, press any key to close it. 
 ``