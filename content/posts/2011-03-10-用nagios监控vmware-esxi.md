---
title: 用nagios监控vmware esxi
author: 阿辉
date: 2011-03-10T16:09:00+00:00
categories:
- Vmware ESXI
tags:
- Vmware ESXI
keywords:
- Vmware ESXI
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
在网上找了个监控vmware esxi的脚本，配置了一下，用起来很不错。

脚本：

http://exchange.nagios.org/directory/Plugins/Operating-Systems/*-Virtual-Environments/VMWare/Check-hardware-running-VMware-ESXi/details

脚本下来后，加参数运行就行了：

<!--more-->

```bash
#./check_esx_wbem.py https://192.168.1.10 root passwd
20110310 01:53:16 Connection to https://192.168.1.10
20110310 01:53:16 Check classe CIM_ComputerSystem
20110310 01:53:16 Element Name = System Board 7:1
20110310 01:53:16 Element Op Status = 0
20110310 01:53:16 Element Name = Add-in Card 11:2
20110310 01:53:16 Element Op Status = 0
20110310 01:53:16 Element Name = localhost.localdomain
20110310 01:53:16 Element Name = Hardware Management Controller (Node 0)
20110310 01:53:16 Element Op Status = 0
20110310 01:53:16 Element Name = Controller 0 (PERC 6/i Integrated)
20110310 01:53:16 Element Op Status = 2
20110310 01:53:16 Check classe CIM_NumericSensor
20110310 01:53:17 Element Name = System Board 1 System Level
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = Power Supply 1 Voltage 1
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = Power Supply 1 Current 1
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 FAN 5 RPM
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 FAN 4 RPM
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 FAN 3 RPM
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 FAN 2 RPM
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 FAN 1 RPM
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = System Board 1 Ambient Temp
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Check classe CIM_Memory
20110310 01:53:17 Element Name = CPU1 Level-1 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = CPU1 Level-2 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = CPU1 Level-3 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = CPU2 Level-1 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = CPU2 Level-2 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = CPU2 Level-3 Cache
20110310 01:53:17 Element Op Status = 0
20110310 01:53:17 Element Name = Memory
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Check classe CIM_Processor
20110310 01:53:17 Element Name = CPU1
20110310 01:53:17 Element Op Status = 2
20110310 01:53:17 Element Name = CPU2
20110310 01:53:17 Element Op Status = 15
20110310 01:53:17 Check classe CIM_RecordLog
20110310 01:53:18 Element Name = IPMI SEL
20110310 01:53:18 Element Op Status = 2
20110310 01:53:18 Check classe OMC_DiscreteSensor
20110310 01:53:19 Element Name = Add-in Card 2 SD vFLash Status 1
20110310 01:53:19 Element Name = Disk Drive Bay 3 ROMB Battery 0: Low
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 3 ROMB Battery 0: Failed
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Cable SAS B 0: Connected
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Cable SAS B 0: Config Error
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Cable SAS A 0: Connected
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Cable SAS A 0: Config Error
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 5: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 4: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 3: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 2: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 1: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Drive Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Drive Fault
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Predictive Failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Hot Spare
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Parity Check In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: In Critical Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: In Failed Array
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Rebuild In Progress
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Disk Drive Bay 1 Drive 0: Rebuild Aborted
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 Power Optimized 0: OEM
20110310 01:53:19 Element Name = System Board 1 Power Optimized 0: Unknown
20110310 01:53:19 Element Name = System Board 1 Power Optimized 0: Unknown
20110310 01:53:19 Element Name = System Board 1 Power Optimized 0: Unknown
20110310 01:53:19 Element Name = System Board 1 Fan Redundancy 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 Intrusion 0: General Chassis intrusion
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 OS Watchdog 0: Timer expired
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 OS Watchdog 0: Hard reset
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 OS Watchdog 0: Power down
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 OS Watchdog 0: Power cycle
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 Riser Config 0: Connected
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 Riser Config 0: Config Error
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Power Supply 1 Status 0: Presence detected
20110310 01:53:19 Element Name = Power Supply 1 Status 0: Failure detected
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Power Supply 1 Status 0: Predictive failure
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Power Supply 1 Status 0: Power Supply AC lost
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Power Supply 1 Status 0: Config Error: Vendor Mismatch
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 Status 0: IERR
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 Status 0: Thermal Trip
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 Status 0: Configuration Error
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 Status 0: Presence detected
20110310 01:53:19 Element Name = Processor 2 Status 0: Throttled
20110310 01:53:19 Element Name = Processor 1 Status 0: IERR
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 Status 0: Thermal Trip
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 Status 0: Configuration Error
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 Status 0: Presence detected
20110310 01:53:19 Element Name = Processor 1 Status 0: Throttled
20110310 01:53:19 Element Name = Disk Drive Bay 1 Presence  0: Present
20110310 01:53:19 Element Name = Disk Drive Bay 1 Presence  0: Absent
20110310 01:53:19 Element Name = Power Supply 2 Presence 0: Present
20110310 01:53:19 Element Name = Power Supply 2 Presence 0: Absent
20110310 01:53:19 Element Name = Power Supply 1 Presence 0: Present
20110310 01:53:19 Element Name = Power Supply 1 Presence 0: Absent
20110310 01:53:19 Element Name = Processor 2 Presence 0: Present
20110310 01:53:19 Element Name = Processor 2 Presence 0: Absent
20110310 01:53:19 Element Name = Processor 1 Presence 0: Present
20110310 01:53:19 Element Name = Processor 1 Presence 0: Absent
20110310 01:53:19 Element Name = System Board 1 Riser1 Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 Riser1 Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 Riser2 Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 Riser2 Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 Stor Adapt Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 Stor Adapt Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 USB Cable Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 USB Cable Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 iDRAC6 Ent Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 iDRAC6 Ent Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 Heatsink Pres 0: Present
20110310 01:53:19 Element Name = System Board 1 Heatsink Pres 0: Absent
20110310 01:53:19 Element Name = System Board 1 1.05 V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 1.0 AUX PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 1.0 LOM PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 1.1 V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 8.0 V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 1.8 PLL PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 1.8 PLL  PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 0.9V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 VTT  0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 VTT  0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 MEM PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 MEM PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 5V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 3.3V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 1.8V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 1.5V PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 0.75 VTT CPU1 PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 0.75 VTT CPU2 PG 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 2 VCORE 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = Processor 1 VCORE 0
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Element Name = System Board 1 CMOS Battery 0: Failed
20110310 01:53:19 Element Op Status = 2
20110310 01:53:19 Check classe VMware_StorageExtent
20110310 01:53:20 Element Name = Drive 0 in enclosure 32 on controller 0 Fw: SN11 - ONLINE
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Drive 1 in enclosure 32 on controller 0 Fw: 3B05 - ONLINE
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Drive 2 in enclosure 32 on controller 0 Fw: KA05 - ONLINE
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Drive 3 in enclosure 32 on controller 0 Fw: 0001 - ONLINE
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Check classe VMware_Controller
20110310 01:53:20 Element Name = Controller 0 (PERC 6/i Integrated)
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Check classe VMware_StorageVolume
20110310 01:53:20 Element Name = RAID 10 Logical Volume 0 on controller 0, Drives(0e32,1e32,2e32,3e32)  - OPTIMAL
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Check classe VMware_Battery
20110310 01:53:20 Element Name = Battery on Controller 0
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Check classe VMware_SASSATAPort
20110310 01:53:20 Element Name = Port 0 on Controller 0
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Port 1 on Controller 0
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Port 2 on Controller 0
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Port 3 on Controller 0
20110310 01:53:20 Element Op Status = 2
20110310 01:53:20 Element Name = Port 4 on Controller 0
20110310 01:53:20 Element Op Status = 15
20110310 01:53:20 Element Name = Port 5 on Controller 0
20110310 01:53:20 Element Op Status = 15
20110310 01:53:20 Element Name = Port 6 on Controller 0
20110310 01:53:20 Element Op Status = 15
20110310 01:53:20 Element Name = Port 7 on Controller 0
20110310 01:53:20 Element Op Status = 15
OK
```
输出量很大，其实是可以关掉的，把脚本内的 verbose = 1改成0就是了。

这个脚本用到了PyWEEM模块，如果机器上没有，需要安装的：

PyWEEM模块主页：http://pywbem.sourceforge.net/
```bash
# wget http://downloads.sourceforge.net/project/pywbem/pywbem/pywbem-0.7/pywbem-0.7.0.tar.gz?r=http%3A%2F%2Fsourceforge.net%2Fprojects%2Fpywbem%2Ffiles%2Fpywbem%2F&ts=1299742557&use_mirror=voxel
# tar -xvzf pywbem-0.7.0.tar.gz
# cd pywbem-0.7.0
# python setup.py build
# python setup.py install
```
 

下面配置到nagios内：

1) 先加命令：

#vi /etc/nagios/objects/commands.cfg

在最后加上：
```
define command{
        command_name    check_esxi
        command_line    /etc/nagios/command/check_esx_wbem.py https://$HOSTADDRESS$ $ARG1$ $ARG2$
        }
```
 

2) 再加服务：

#vi linuxhosts.cfg

也在最后加上
```
define service{
        use                             generic-service         ; Name of service template to use
        host_name                       esxi
        service_description             check_esxi
        check_command                   check_esxi!root!passwd
        notifications_enabled           1
        }
```

3) 重载nagios
```
#service nagios reload
```