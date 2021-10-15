{{>toc}}
# Alius watchdog software design

# 0. History

## Version 0.1

* Update the general module and description.
* Merge same cases together(model 1-3).
* Provide range of timeout.
* Change target core NOT need to notify M0 to reset, reset always control by master core or M0.

## Version 0.2 draft
* Add AVS-VP6-WDT timeout reset interrupt process description.

# 1. Overview

## 1.1 Overview
The Watchdog Timer (WDT) can be used to prevent system lockup that may be caused by conflicting parts or programs in SoC. An interrupt or system reset will be generated when decrementing counter reach zero. The generated interrupt is passed to an interrupt controller. The generated reset is passed to a reset controller, which in turn generates a reset for the components in the system.

## 1.2 Reference
Alius_Hardware_Spec_V0p42.pdf
DW_apb_wdt_databook.pdf

# 2. SYNOPSYS DesignWare watchdog IP

## 2.1 Block
<img style="width:40%" src="dw_wdt.png"/>

## 2.2 Operation flow
<img style="width:40%" src="dw_wdt_operation_flow.png"/>

Reload counter is where the software need to feed watchdog

## 2.3 Register
<img style="width:40%" src="dw_wdt_register.png"/>

## 2.4 Timeout period
Timeout period can config by WDT_TORR register,t = 2^(16 + i) Clocks ( For i = 0 to 15 ), 
Default clock is 24M. i with	watchdog reset time as below
0	2.73*2 ms
1	5.64*2 ms
2	10.9*2 ms
3	21.8*2 ms
4	43.7*2 ms
5	81.4*2 ms
6	175*2 ms
7	350*2 ms
8	699*2 ms
9	1.4*2 s
10	2.8*2 s
11	5.6*2 s
12	11.2*2 s
13	22.4*2 s
14	44.7*2 s
15	89.5*2 s

# 3. Alius watchdog Hardware design
## 3.1 Hardware config
### 3.1.1 Overview
Alius has independent ten WDT for each CPU/MCU and DSP.
• ACS-A32-WDT*2
• LCS-A32-WDT
• M33-WDT
• AVS-VP6-WDT
• LVS-VP6-WDT
• HiFi3z_0-WDT
• HiFi3z_1-WDT
• F1-WDT
• M0-WDT

Refer to 3.1.3 Watchdog Scope for each target core and master core.

### 3.1.2 Feature
• 32-bit watchdog counter
• First timeout occurs, it generate a timeout interrupt, a second timeout occurs then generate a reset interrupt.
better. For example the first one is timeout interrupt, and second one is reset interrupt. </span> 
• Programmable timeout range
• Support for external timer clock(not use apb clock) for counter.
• Each watchdog has independent clk enable signal and reset signal, control by M0

### 3.1.3 Watchdog Scope
<img style="width:40%" src="alius_wdt_scope1.png"/>
<img style="width:40%" src="alius_wdt_scope2.png"/>

AVS-VP6-WDT timeout reset interrupt can handle by LP.A32 or HP.A32.
When HP.A32 power off, the interrupt controller will route the watchdog timeout reset interrupt (SPI type) to LP.A32, 
and LP.A32 will send the request to power management unit for reset VP6 and watchdog. 

# 4. Alius watchdog software design
## 4.1 Overview
{{drawio(wdt_overview.xml,zoom=80)}}

**target core** : which is monitor by watchdog. it should feed watchdog, otherwise watchdog timeout will occur.
**master core** : which is process target core watchdog timeout interrupt, it will do something process if need. then notify PMU to reset target core and watchdog.
**PMU** : in Alius it is M0, it is responsible for reset target core and watchdog.

Tips：
If master core hangs, master has its own watchdog, master core will be reset by its own watchdog


## 4.2 Watchdog software design
Different watchdog processing strategies are different, divided into three types

### 4.2.1 Model1 M0 only(LAS-WDT-M0)
{{drawio(model1.xml)}}

Flow:
1. M0 firmware power up, set watchdog timeout time and enable watchdog through apb register. start task to feed watchdog.
2. If watchdog stage1 timeout, watchdog generate interrupt to M0 NVIC, M0 software dump exception info and reset SOC.
3. If watchdog stage2 timeout, watchdog generate hardware signal to reset SOC, Software do nothing here, not have any software logic.<br>

Task:
M0 firmware: 
* init and enable watchdog
* feed watchdog
* process stage1 timeout interrupt, dump exception info and reset SOC

### 4.2.2 Model2 target core + M0 (LAS-WDT-M33/LPS-WDT-LCS)
{{drawio(model2.xml)}}

Flow:
1. target core(M33/LP A32) os power up, set watchdog timeout time and enable watchdog through apb register. start task to feed watchdog.
2. If watchdog stage1 timeout, watchdog generate interrupt to target core, target core software dump exception info.
3. If watchdog stage2 timeout, watchdog generate interrupt(reset) notify to M0, M0 reset target core and corresponding watchdog.

Task:
target core: 
* init and enable corresponding watchdog
* feed watchdog
* process stage1 timeout interrupt, dump exception info

m0 firmware:
* process watchdog stage2 timeout interrupt, reset target core and corresponding watchdog

### 4.2.3 Model3 target core + master core + M0 (LAS-WDT-F1/LPS-WDT-HiFi3z/LPS-WDT-VP6/APS-WDT-ACS)
{{drawio(model3.xml)}}

Flow:
1. target core(DSP/HP A32) os power up, set watchdog timeout time and enable watchdog through apb register. start task to feed watchdog.
2. (option)If watchdog stage1 timeout, watchdog generate interrupt to target core, target core software dump exception info.
3. If watchdog stage2 timeout, watchdog generate interrupt(reset) notify to master core(M33/LP A32), master core do something process and notify M0 to reset, M0 reset target core and corresponding watchdog.

Task:
target core: 
* init and enable corresponding watchdog
* feed watchdog
* (option)process stage 1 timeout interrupt, dump exception info

master core:
* Process watchdog stage2 timeout, if need, do something, then notify M0 to reset (through mailbox)

m0 firmware:
* process reset request from mailbox, reset target core and corresponding watchdog

### 4.2.4 Power
{{drawio(hp_a32_wdt2.xml, zoom=60, page=0)}}
HP A32 do power operate, the corresponding watchdog also should do power operate:
If A32 core hotplug power down, corresponding watchdog should be reset.
If A32 core standby, corresponding watchdog should be pause.

### 4.2.4 System shutdown
While system shutdown, the dog feeding program will be stopped，and the M0 will power down system. If the time between the two exceeds the watchdog timeout time, the watchdog will be triggered, target core software should ensure that this time does not exceed the watchdog timeout time.
