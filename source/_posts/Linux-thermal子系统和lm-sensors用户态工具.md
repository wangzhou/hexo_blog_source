---
title: Linux thermal子系统和lm_sensors用户态工具
tags:
  - Linux内核
description: 本文分析Linux thermal子系统的现状，以及可能与之配套使用的lm_sensors用户态工具的软件构架。提供给写thermal驱动的同学可以参考。
abbrlink: e597e2ee
date: 2021-06-27 17:59:41
---

1. Linux thermal驱动
--------------------

   Linux thermal是一个内核和温度检测控制有关的驱动子系统，他的位置在drivers/thermal/*.
   相关的内核头文件在include/linux/thermal.h。具体的设备驱动需要向thermal框架注册
   thermal_zone_device, thermal框架会在thermal_zone_device里封装一个device向系统
   注册，通过这个device向用户态暴露一组sysfs属性文件。用户态可以通过这组文件设置相关
   参数、获取相关信息。

   在我的笔记本上，这组属性文件大概是这样的：
```
wangzhou@kllp05:/sys/class$ tree thermal/thermal_zone0
thermal/thermal_zone0
├── available_policies
├── device -> ../../../LNXSYSTM:00/LNXSYBUS:01/LNXTHERM:00
├── emul_temp
├── integral_cutoff
├── k_d
├── k_i
├── k_po
├── k_pu
├── mode
├── offset
├── passive
├── policy
├── power
│   ├── async
│   ├── autosuspend_delay_ms
│   ├── control
│   ├── runtime_active_kids
│   ├── runtime_active_time
│   ├── runtime_enabled
│   ├── runtime_status
│   ├── runtime_suspended_time
│   └── runtime_usage
├── slope
├── subsystem -> ../../../../class/thermal
├── sustainable_power
├── temp
├── trip_point_0_temp
├── trip_point_0_type
├── type
└── uevent
```
   这里只是展示了thermal_zone0, 当然一个系统里可以多个这样的设备。
   
   实际上一个具体的驱动和thermal子系统的关系可以通过三个对象去描述，一个就是
   这里的thermal_zone_thermal，它表示测量温度的sensor；还可以注册这个sensor所在
   温度管理域的降温设备；有了降温设备，还可以注册相应的温度调节策略。（to do: ...）

   实际上，你要是只想读温度出来，只注册一个thermal_zone_thermal并提供其中一个
   获取温度的回调函数的实现就可以了：thermal_zone_device_ops->.get_temp。这个函数
   会被/sys/class/thermal/<dev>/temp的show函数调用，从而显示测量的温度。

   thermal子系统还会根据注册thermal_zone_device时的参数，把设备的信息通过
   /sys/class/hwmon子系统暴露出来。如果你不带注册参数，thermal子系统默认会通过
   /sys/class/hwmon暴露信息, 比如这样：
```
wangzhou@kllp05:/sys/class$ tree hwmon/hwmon0
hwmon/hwmon0
├── name
├── power
│   ├── async
│   ├── autosuspend_delay_ms
│   ├── control
│   ├── runtime_active_kids
│   ├── runtime_active_time
│   ├── runtime_enabled
│   ├── runtime_status
│   ├── runtime_suspended_time
│   └── runtime_usage
├── subsystem -> ../../../../class/hwmon
├── temp1_crit
├── temp1_input
└── uevent
```
   temp1_input的show函数会最终调用到驱动里的.get_temp。


2. lm_sensors用户态工具的使用
-----------------------------

  lm_sensors可以读取系统上sensor的信息。它的源代码可以在这里下载到：
  https://github.com/lm-sensors/lm-sensors.git

  粗略从代码上看，它使用的是hwmon接口提供的信息。
  
  在ubuntu系统上你可以使用如下命令简单尝试下lm_sensors:
```
sudo apt-get install lm-sensors

/* this is a Perl script in /usr/sbin/ */
sudo sensors-detect
输入这个命令后，一路YES。

wangzhou@kllp05:~/notes$ sensors
iwlwifi-virtual-0
Adapter: Virtual device
temp1:        +37.0°C  

thinkpad-isa-0000
Adapter: ISA adapter
fan1:           0 RPM

acpitz-virtual-0
Adapter: Virtual device
temp1:        +30.0°C  (crit = +128.0°C)

coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +33.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +31.0°C  (high = +100.0°C, crit = +100.0°C)
Core 1:        +33.0°C  (high = +100.0°C, crit = +100.0°C)

pch_skylake-virtual-0
Adapter: Virtual device
temp1:        +30.0°C  
```

3. lm_sensors分析
-----------------

  (to do: sensors-detect, sensors, sensord, config...)

4. Linux thermal和lm_sensors的关系
----------------------------------

  如上，现在的lm_sensors使用的是hwmon接口获取信息。
  (to do: ...)
