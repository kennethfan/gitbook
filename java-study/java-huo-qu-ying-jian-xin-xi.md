---
description: 背景
---

# java获取硬件信息

最近项目中有功能需要获取服务器的一些硬件信息，调研了下oshi，这里做个记录



demo

```bash
git clone https://github.com/oshi/oshi.git 
cd oshi
./mvnw test-compile -pl oshi-core exec:java -Dexec.mainClass="oshi.SystemInfoTest" -Dexec.classpathScope="test"
```

输出结果如下

```
Apple macOS 12.6.8 (Monterey) build 21G725
Booted: 2023-10-24T01:16:53Z
Uptime: 22 days, 06:39:44
Running without elevated permissions.
Sessions:
 kenneth, console, 2023-10-24 09:16
 kenneth, ttys000, 2023-10-24 09:16
 kenneth, ttys002, 2023-11-01 16:53
 kenneth, ttys006, 2023-11-03 10:49
 kenneth, ttys008, 2023-11-13 16:36
 kenneth, ttys007, 2023-11-07 10:48
 kenneth, ttys009, 2023-11-14 09:40
System: manufacturer=Apple Inc., model=MacBookPro11,4, serial number=C02WD2QGG8WN, uuid=217F1A89-A485-56E4-8F96-80FAB192F04D
 Firmware: manufacturer=Apple Inc., name=boot.efi, description=EFI64, version=481.0.0.0.0, release date=01/12/2023
 Baseboard: manufacturer=Apple Inc., model=Mac-06F11FD93F0323C5, version=1.0, serial number=C02WD2QGG8WN
Intel(R) Core(TM) i7-4770HQ CPU @ 2.20GHz
 1 physical CPU package(s)
 4 physical CPU core(s)
 8 logical CPU(s)
Identifier: Intel64 Family 6 Model 70 Stepping 1
ProcessorID: bfebfbff00040661
Microarchitecture: Haswell (Client)
 Topology:
  LogProc  P/E Proc  Pkg NUMA PGrp
      0,1    P    0    0    0    0
      2,3    P    1    0    0    0
      4,5    P    2    0    0    0
      6,7    P    3    0    0    0
 Caches:
  ProcessorCache [L3 Unified, cacheSize=6291456, unknown associativity, lineSize=64]
  ProcessorCache [L2 Unified, cacheSize=262144, 8-way associativity, lineSize=64] (per core)
  ProcessorCache [L1 Instruction, cacheSize=32768, unknown associativity, lineSize=64] (per core)
  ProcessorCache [L1 Data, cacheSize=32768, unknown associativity, lineSize=64] (per core)
Physical Memory:
 Available: 5.6 GiB/16 GiB
Virtual Memory:
 Swap Used/Avail: 3.9 GiB/5 GiB, Virtual Memory In Use/Max=14.3 GiB/21 GiB
Physical Memory:
 Bank label: BANK 0/DIMM, Capacity: 8 GiB, Clock speed: 1.6 GHz, Manufacturer: 0x802C, Memory type: DDR3
 Bank label: BANK 1/DIMM, Capacity: 8 GiB, Clock speed: 1.6 GHz, Manufacturer: 0x802C, Memory type: DDR3
Context Switches/Interrupts: 0 / 0
CPU, IOWait, and IRQ ticks @ 0 sec:[44476225, 0, 17022524, 336836000, 0, 0, 0, 0]
CPU, IOWait, and IRQ ticks @ 1 sec:[44476374, 0, 17022554, 336836625, 0, 0, 0, 0]
User: 18.5% Nice: 0.0% System: 3.7% Idle: 77.7% IOwait: 0.0% IRQ: 0.0% SoftIRQ: 0.0% Steal: 0.0%
CPU load: 22.3%
CPU load averages: 5.41 4.38 4.07
CPU load per processor: 51.0% 4.0% 41.0% 4.0% 34.7% 4.0% 34.7% 4.9%
Vendor Frequency: 2.2 GHz
Max Frequency: 2.2 GHz
Current Frequencies: 2.2 GHz, 2.2 GHz, 2.2 GHz, 2.2 GHz, 2.2 GHz, 2.2 GHz, 2.2 GHz, 2.2 GHz
My PID: 60031 with affinity 11111111
My TID: 2 with details OSThread [threadId=2, owningProcessId=60031, name=, state=SLEEPING, kernelTime=2260, userTime=12720, upTime=237981, startTime=1700034760880, startMemoryAddress=0x0, contextSwitches=0, minorFaults=0, majorFaults=0]
Processes: 681, Threads: 1619
   PID  %CPU %MEM       VSZ       RSS Name
 60031  20.1  3.0  38.4 GiB 488.5 MiB java
 57591  10.1 18.9  40.9 GiB   3.0 GiB idea
 58388   7.1  0.6  33.3 GiB  93.2 MiB QQMusic
 57531   7.0  0.8   1.1 TiB 125.0 MiB Google Chrome Helper (Renderer)
 59818   4.4  2.0   1.1 TiB 319.9 MiB Google Chrome Helper (Renderer)
```

参考

[https://github.com/oshi/oshi](https://github.com/oshi/oshi)
