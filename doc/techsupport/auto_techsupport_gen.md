# Auto Techsupport Enhancement #
#### Rev 1.0

## Table of Contents
  * [Revision](#revision)
  * [About this Manual](#about-this-manual)
  * [1. Overview](#1-overview)
  * [2. High Level Requirements](#2-high-level-requirements)
  * [3. Core Dump Generation in SONiC](#3-core-dump-generation-in-sonic)
  * [4. Schema Additions](#4-schema-additions)
  * [5. CLI Enhancements](#5-cli-enhancements)
  * [6. Design](#6-design)
      * [6.1 Event trigger for Core-dump generation](#61-Event-trigger-for-Core-dump-generation)
      * [6.2 Monitor Techsupport creation](#62-Monitor-Techsupport-Creation)
      * [6.3 auto_techsupport_gen script](#63-auto_techsupport_gen-script)
      * [6.4 Adding these services to SONiC](#64-Adding-these-services-to-sonic)


### Revision  
| Rev |     Date    |       Author       | Change Description          |
|:---:|:-----------:|:-------------------------|:----------------------|
| 1.0 | 06/14/2021  | Vivek Reddy Karri        | Auto Invocation of Techsupport, triggered by a core dump       |


## About this Manual
This document describes the details of the system which facilitates the auto techsupport invocation support in SONiC. The auto invocation is triggered when any process across the dockers or the host crashes and a core dump is generated.

## 1. Overview
Currently, techsupport is run by invoking `show techsupport` either by orchestration tools like Jenkins or manually. The techsupport dump also collects any core dump files available in the `/var/core/` directory.

However if the techsupport invocation can be made event-driven based on core dump generation, that would definitely improve the debuggability. That is the overall idea behind this HLD. All the high-level requirements are summarized in the next section

## 2. High Level Requirements
* Techsupport invocation should also be made event-driven based on core dump generation
* This capability should be made optional and is disabled by default
* Users should have the abiliity to configure this capability.

## 3. Core Dump Generation in SONiC
In SONiC, the core dumps generated from any process crashes across the dockers and the base host are directed to the location `/var/core` and will have the naming format `/var/core/*.core.gz`. 
The naming format and compression is governed by the script `/usr/local/bin/coredump-compress`.

## 4. Schema Additions

#### Config DB
```
key = "AUTO_TECHSUPPORT|global"
state = enabled|disabled; 
cooloff = 300;    # Minimum Time in seconds, between two successive techsupport invocations by the script.
```

#### State DB
```
key = "AUTO_TECHSUPPORT|global"
last_techsupport_run = 0; # Monotonic time in seconds relative to the latest techsupport run   
enabled = yes|no;
core_file_list = "<*.core.gz>;<*.core.gz>"; List of the core files inside the /var/core/ folder 
                                     Eg: "python3.15678876.168.core.gz;orchagent.145678765.182.core.gz;...."
```

## 5. CLI Enhancements.

### config cli

`config auto-techsupport state <enabled/disabled>`

`config auto-techsupport cooloff <seconds>`

### show cli

```
admin@sonic:~$ show auto-techsupport 
+----------------+----------------+-----------------------------------+
|     Enabled    |  Cooloff (sec) |       Last TechSupport Run        |
+================+================+===================================+
|       Yes      |      300       |   Tue 15 Jun 2021 08:09:59 PM UTC |
+----------------+----------------+-----------------------------------+
```

## 6. Design

### 6.1 Event-trigger for Core-dump generation
To Monitor and respond for the file-change events in `/var/core/`, a systemd path unit ([systemd path unit](https://www.freedesktop.org/software/systemd/man/systemd.path.html)) will be used. This unit will start a corresponding systemd service, which inturn invokes the python script `/usr/local/bin/auto_techsupport_gen` and it handles the heavylifting of invoking techsupport and other tasks. More on the script in section 6.3 

#### coredump-monit.path
```
[Unit]
Description=Triggers the coredump-monit services accordingly when a coredump is found.
After=database.service
Requires=database.service

[Path]
PathChanged=/var/core/
Unit=coredump-monit.service

[Install]
WantedBy=multi-user.target
```

#### coredump-monit.service
```
[Unit]
Description=Invokes the auto_techsupport_gen script when triggered by the coredump-monit.path
After=database.service
Requires=database.service

[Service]
Type=simple
ExecStart=/usr/local/bin/auto_techsupport_gen core

[Install]
WantedBy=multi-user.target
```

### 6.2 Monitor Techsupport creation
The script will use the last_techsupport_run field in the State DB to determine whether to run techsupport based on the cooloff period configured by the user. To have the last_techsupport_run upto date, techsupport-monit.{path, service} units is used.


#### techsupport-monit.path
```
[Unit]
Description=Triggers the auto_techsupport_gen services when a techsupport dump is found.
After=database.service
Requires=database.service

[Path]
PathChanged=/var/dump/
Unit=techsupport-monit.service

[Install]
WantedBy=multi-user.target
```

#### techsupport-monit.service
```
[Unit]
Description=Invokes the auto_techsupport_gen script when triggered by the techsupport-monit.path
After=database.service
Requires=database.service

[Service]
Type=simple
ExecStart=/usr/local/bin/auto_techsupport_gen techsupport

[Install]
WantedBy=multi-user.target
```

Note: All of these will have strict ordering dependency on database.service and not swss or sonic.target, because the crashes might occur during the swss/syncd bringup etc. And for this to be captured the service should be active before the start of these services. 

### 6.3 auto_techsupport_gen script

As seen in the techsupport-monit.service & coredump-monit.service Unit descriptions, the script follows two separate flows based on the argument provided.  When invoked with `techsupport` argument, the script updates the `last_techsupport_run` field in the State DB. 

On the other hand, when invoked with `core` argument, the script first checks if this feature is enabled by the user. The Script then checks for any diff between `core_file_list` field in the State DB and the file system. If any diff is found, it updates the State Db entry and moves forward. The script finally checks the `last_techsupport_run` field in the State DB and only when the cooloff period has passed, the script invokes the techsupport.

**Note: The last_techsupport_run value doesn't persist across reboots, since monotonic time is used. The field will be empty after reboot (including warm-boot)**. 

### 6.4 Adding these services to SONiC

These will be added to `target/debs/buster/sonic-host-services-data_1.0-1_all.deb`.




