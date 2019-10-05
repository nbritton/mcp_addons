# Mirantis MCP - Dell PowerEdge Platform Integration:

The included yaml files in this directory will integrate Dell's platform management tools for PowerEdge servers into MCP. Specifically, these will pull in Dell's OpenManage suite, Dell EMC System Update, and PERCCLI utilities to facilitate advanced platform management and health monitoring capabilities.

#### Included are:

- ##### Dell EMC OpenManage Server Administrator (OMSA):

  | Package:                 | Description:                             |
  | ------------------------ | ---------------------------------------- |
  | srvadmin-cm              | Inventory collector (IC)                 |
  | srvadmin-base            | Base OMSA, no web server                 |
  | srvadmin-idrac           | Common components to manage iDRAC        |
  | srvadmin-idrac7          | RACADM for iDRAC7                        |
  | srvadmin-idracadm8       | RACADM for iDRAC8 and above              |
  | srvadmin-webserver       | Web Interface (not installed by default) |
  | srvadmin-storageservices | RAID Management                          |
  | syscfg                   | SYSCFG for BIOS Configuration            |
  | raidcfg                  | RAIDCFG for Storage Configuration        |
  | dcism                    | iDRAC Service Module                     |

- ##### Dell EMC System Update:

  | Package:                 | Description:                             |
  | ------------------------ | ---------------------------------------- |
  | dsu                      | DSU for Firmware Management              |
  
- ##### PERCCLI:

  | Package: | Description:                                |
  | -------- | ------------------------------------------- |
  | perccli  | CLI for managing PowerEdge RAID Controllers |



#### Dell EMC System Update:

The DSU utility is the primary tool for doing firmware updates to Dell PowerEdge servers, this gives you the ability to update firmware inband via the Linux operating system, rather then having to do it through the iDRAC. This method can be scripted and executed in parallel via salt. Updates can be applied at any time and they will take effect whenever the system is subsequently rebooted.

To pull down the Ubuntu DSU packages you also need:
https://github.com/nbritton/get-dell-dsu-deb-pkgs

To build the Ubuntu PERCCLI packages you also need:
https://github.com/nbritton/make-perccli-deb-pkgs

For interacting with iDRACs (pre-deployment) in a scripted and/or parallel fashion see also:
https://github.com/nbritton/mcp_addons/blob/master/maas_machines_yaml_generator/idrac-exec

#### With this initial release, the following Dell R630 system firmware versions have been validated with MCP:

- iDRAC / Lifecycle Controller: 2.63.60.61
- BIOS: 2.9.1
- PERC H730P Mini: 25.5.5.0005
- SAS Expander (BP13G+EXP 0:1): 3.35
- Toshiba AL14SEB060N: DM06
- HGST HUC101860CSS204: FJ39
- Seagate ST600MM0088: TT31
- Samsung MZ7KM1T9HMJP0D3: GD57
- Intel X520 NIC: 18.5.18 (Do not upgrade to 18.8.9 without first unplugging the machine to drain flea power.)
- Dell 32 Bit uEFI Diagnostics: 4239A36
- OS Collector: 2.1
- System CPLD: 1.0.1
- iDRAC Service Module: 3.2.0.1

Dell System Update ("DSU") is the utility used to update the firmware on Dell machines.

The recommend way to install and manage the DSU utility is by adding the dsu.yml reclass configuration file to the salt model, and running:
```bash
http://www.exabit.io/ubuntu/get-dell-dsu-deb-pkgs | bash
```
from  cfg01:/var/www/html/dell/dsu/, this script will extract the DSU .deb packages from the .BIN files and setup a local HTTP repository for them on the cfg01 node. Once get-dell-dsu-deb-pkgs is finished you can then add the class include into the model for the machines you want to install this on.
```yaml
paramters:
  reclass:
    storage:
      node:
        openstack_compute_dell:
          name: ${_param:openstack_compute_hostname}<<count>>
          domain: ${_param:cluster_domain}
          classes:
          - cluster.${_param:cluster_name}.openstack.compute
          - cluster.${_param:cluster_name}.infra.tools.dell.r630
```

Then once that is complete just run the linux.system state and salt will do the rest of the work.

To update the firmware, the following command will update everything except for network firmware and the iDRAC service module:
```bash
dsu --non-interactive --category=AS,BI,CE,DD,DI,DR,ES,FC,FW,SA,SE,SF,SV,TH,XP
```

##### To install DSU manually (managed by dpkg, outside of the apt package repository) run the following commands:
```bash
wget https://downloads.dell.com/FOLDER05327755M/1/Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN;
chmod +x Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN;
./Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN -q;
```

#### This can be scripted with salt using the following command:
```bash
salt '<dell-machine>' cmd.run 'export TERM=xterm; export file=Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN; wget -q https://downloads.dell.com/FOLDER05327755M/1/${file}; chmod +x ${file}; ./${file} -qf; rm ${file};'
```

#### Very Important:
**Intel X520 network interface firmware cannot be upgraded with DSU at this time**, because there is an issue within the Intel X520 firmware version 18.8.9, do not upgrade to that version without first ensuring you have physical access to disconnect the machine from mains power. Updating past version 18.5.18 requires a complete power cycle of the machine, which means the machine has to be unplugged from the mains to drain residual flea power, this is necessary to trigger the NICs to reset (normally it stays online even when the machine is turned off for Wake-on-LAN functionality) so that they can load the new data structures that were added in the new firmware. If this step is not followed the NICs will not come online after the update. It is possible to rollback if a machine was accidentally upgrade to 18.8.9, and can't be physically unplugged, by reapplying the 18.5.18 firmware[1]. Use the iDRAC LifeCycle controller to downgrade the firmware to 18.5.8, and through experience I've found that one usually will need to repeat this step at least two times for the change to take effect. It also may be helpful to trigger an On/Off power cycle via the iDRAC. If that doesn't work a couple other methods exist, but these require transferring the DUP BIN to the host operating system via the iDRAC Virtual Media device.

[1]: https://downloads.dell.com/FOLDER05152341M/2/Network_Firmware_3XJH0_WN64_18.5.18_A00_01.EXE

To update the Intel X520 NIC firmware manually, to version 18.5.18, run the following commands:

```bash
wget https://downloads.dell.com/FOLDER05152339M/1/Network_Firmware_3XJH0_LN_18.5.18_A00.BIN;
bash ./Network_Firmware_3XJH0_LN_18.5.18_A00.BIN -qf;
```

Note: In machines with multiple Intel X520 cards, it is usually safe to disregard any error messages that say the update failed, this is typically related to a bug in the DUP scripts that causes this erroneous message to be displayed. If after rebooting, the device does not come back online reapply the firmware update (via iDRAC, at least once, possibly twice) and reboot, this almost always will fix it.

To apply the Intel X520 NIC firmware using salt, you must set the TERM variable, for example:
```bash
salt -E '.*(kvm|cmp).*' cmd.run 'export TERM=xterm; export file=Network_Firmware_3XJH0_LN_18.5.18_A00.BIN; wget https://downloads.dell.com/FOLDER05152339M/1/${file}; chmod +x ${file}; ./${file} -q;'
```

The Dell SysCfg utility configures BIOS settings, for example:
```bash
/opt/dell/toolkit/bin/syscfg --ProcX2Apic=Enabled --IntegratedNetwork1=DisabledOs --ProcVirtualization=Enabled --IoatEngine=Enabled --SysProfile=PerfOptimized --BootSeqRetry=Enabled
```

For Dell PERC H730P controllers with SATA SSDs, Write-Through, Read Ahead, Caching I/O Policy, and Physical Drive Cache Enabled offers the highest performance, and can be set using the following command:
```bash
perccli /c0 /v1 set wrcache=WT rdcache=RA iopolicy=Cached pdcache=On;
```

Enforce cache policy on all RAID volumes with Samsung SSD drives (Model MZ7KM...):
```bash
salt -E '.*(kvm|cmp).*' cmd.run 'test -x /usr/sbin/perccli || exit 1; for volume in $(perccli /c0 show all | awk '"'"'/MZ7KM/ {print $4}'"'"' | sort -u); do perccli /c0 /v${volume} set wrcache=WT rdcache=RA iopolicy=Cached pdcache=On; done'
```
