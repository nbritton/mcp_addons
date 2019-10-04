The included yaml files here will intigrate Dell's platform management tools for PowerEdge servers into MCP. These will pull in Dell's OpenManage suite, Dell System Update, and PERCCLI utilities to allow advanced hardware life-cycle management and hardware health monitoring.

The Dell System Update utility is the primary tool for doing firmware updates to Dell PowerEdge servers, this offers the ability to update firmware inband via the Linux operating system rather then having to do it through the iDRAC. This method can be scripted and executed in parallel via salt. Updates can be applied at any time and will take effect whenever the system is subsequently rebooted.

To pull down the Ubuntu DSU packages you also need:
https://github.com/nbritton/get-dell-dsu-deb-pkgs

To build the Ubuntu PERCCLI packages you also need:
https://github.com/nbritton/make-perccli-deb-pkgs

For interacting with iDRACs (pre-deployment) in a scripted / parrallel fashion see also:
https://github.com/nbritton/mcp_addons/blob/master/maas_machines_yaml_generator/idrac-exec


For Dell PERC H730P controllers with SATA SSDs, Write-Through, Read Ahead, Caching I/O Policy, and Pyshical Drive Cache Enabled offers the highest performance, and can be set using the following command:

perccli /c0 /v1 set wrcache=WT rdcache=RA iopolicy=Cached pdcache=On;

# Enforce cache policy on all RAID volumes with Samsung SSD drives (Model MZ7KM...):
salt -E '.*(kvm|cmp).*' cmd.run 'test -x /usr/sbin/perccli || exit 1; for volume in $(perccli /c0 show all | awk '"'"'/MZ7KM/ {print $4}'"'"' | sort -u); do perccli /c0 /v${volume} set wrcache=WT rdcache=RA iopolicy=Cached pdcache=On; done'


==Currently, the following Dell R630 system firmware versions have been validated with MCP:==
# iDRAC / Lifecycle Controller: 2.63.60.61
# BIOS: 2.9.1
# PERC H730P Mini: 25.5.5.0005
# SAS Expander (BP13G+EXP 0:1): 3.35
# Toshiba AL14SEB060N: DM06
# HGST HUC101860CSS204: FJ39
# Seagate ST600MM0088: TT31
# Samsung MZ7KM1T9HMJP0D3: GD57
# Intel X520 NIC: 18.5.18 (Do not upgrade to 18.8.9 without first unplugging the machine to drain flea power.)
# Dell 32 Bit uEFI Diagnostics: 4239A36
# OS Collector: 2.1
# System CPLD: 1.0.1
# iDRAC Service Module: 3.2.0.1

# Dell System Update ("DSU") is the utility used to update the firmware on Dell machines.

# The recommend why to install and manage DSU packages is by adding the dsu.yml reclass configuration file to the salt model and running "http://www.exabit.io/ubuntu/get-dell-dsu-deb-pkgs | bash" from  cfg01:/var/www/html/dell/dsu/, this will extract the DSU .deb packages from the .BIN files and setup a local http repository for them on the cfg01 node. Once get-dell-dsu-deb-pkgs is finished you can then add the class include into the model for the machines you want to install this on. Then just run the linux.system state and salt will do the rest of the work.

# To install DSU manually (managed by dpkg, outside of the apt package repository) run the following commands:
wget https://downloads.dell.com/FOLDER05327755M/1/Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN;
chmod +x Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN;
./Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN -q;

# This can be scripted with salt using the following command.
salt '<dell-machine>' cmd.run 'export TERM=xterm; export file=Systems-Management_Application_FT56W_LN64_1.6.0_A00.BIN; wget -q https://downloads.dell.com/FOLDER05327755M/1/${file}; chmod +x ${file}; ./${file} -qf; rm ${file};'

# To update the firmware, the following command will update everything except for network firmware and iDRAC service module:
dsu --non-interactive --category=AS,BI,CE,DD,DI,DR,ES,FC,FW,SA,SE,SF,SV,TH,XP

=Very Important=
Intel X520 network firmware cannot be upgraded with DSU at this time because there is a bug within the Intel X520 firmware version 18.8.9, do not upgrade to that version without ensuring you have psyhical access to fully disconnect all power to the system. Updating past version 18.5.18 will require a full power cycle of the machine, which involves unplugging the machine to drain residual flea power, this is necessary to force the NIC cards to reset so that they can load the new firmware. If this step is not followed the NICs will be in a down state within the operating system. It is possible to roll back if a machine was accidently upgrade to 18.8.9, using the 18.5.18 firmware use the iDRAC LifeCycle controller to downgrade the firmware to 18.5.8, this process has to be done a miniumiun of two times for the NICs to start working again, if that doesn't work there are other methods.

# To update the Intel X520 NIC firmware manually, to version 18.5.18, run the following commands:

file=Network_Firmware_3XJH0_LN_18.5.18_A00.BIN;
wget https://downloads.dell.com/FOLDER05152339M/1/${file};
chmod +x ${file};
./${file} -q;

# Note: In machines with multiple X520 cards, ususally it is safe to disregard any error messages that say the update failed, this is typically presumed to be the result of an erroneous message cause by a software bug.

# To apply using salt, setting the TERM variable is necessary.
salt -E '.*(kvm|cmp).*' cmd.run 'export TERM=xterm; export file=Network_Firmware_3XJH0_LN_18.5.18_A00.BIN; wget https://downloads.dell.com/FOLDER05152339M/1/${file}; chmod +x ${file}; ./${file} -q;'

# Dell SysCfg utility configures BIOS settings, run the following command to configure BIOS settings:
/opt/dell/toolkit/bin/syscfg --ProcX2Apic=Enabled --IntegratedNetwork1=DisabledOs --ProcVirtualization=Enabled --IoatEngine=Enabled --SysProfile=PerfOptimized --BootSeqRetry=Enabled
