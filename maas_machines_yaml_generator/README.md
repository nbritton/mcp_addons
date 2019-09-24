# Purpose:

This suite of scripts will generate the maas machine yaml definitions for physical MCP nodes,
it will create a separate yaml file for each machine. Input requires a machine table list to be
provided as an argument to the main script.

```
# The machine table is expected to be a space separated list with the following values:
# column 1: BMC IP Address
# column 2: Machine Serial Number
# column 3: Hostname (should be in lowercase format)
# column 4: Mac Address for NIC 1, interface 1.
# column 5: Mac Address for NIC 1, interface 2.
# column 6: Mac Address for NIC 1, interface 1.
# column 7: Mac Address for NIC 1, interface 2.
```

#  Instructions:

1. Create maas_machines directory in the reclass model on cfg01:

   ```
   mkdir /srv/salt/reclass/classes/cluster/*/infra/maas_machines;
   ```

2. Copy the scripts contained here to maas_machines and make them executable.

3. Gather together a list of the IP addresses for all of Baseboard Management Controllers (i.g. Dell iDRAC, Cisco CIMC) that you want to collect information about. The DTMF Redfish RESTful API must be enabled, requires Dell iDRAC 7+ with firmware v2.40.40.40, or Cisco CIMC v4.0(2h). Save this file to bmc-ip-list.txt or something like that.

4. Using the bmc-ip-list.txt, pass that as an argument to the redfish-get-machine-table script:

   ```
   ./redfish-get-machine-table bmc-ip-list.txt > machine-table.csv;
   ```

5. Sometimes hostnames returned by the BMC are capitalized, these should be converted to lower case:

   ```
   ./convert-hostname-to-lowercase machine-table.csv > lowercase-machine-table.csv;
   ```

6. After your machine table is the way you want it then use the maas-generate-machine-yaml script to generate the yaml machine definitions. For each node it will generate an individual yaml file, the yaml file will be named using the hostname provided in the machine-table.

   ```
   ./maas-generate-machine-yaml lowercase-machine-table.csv;
   ```

7. After you have all the machine definition yaml files created then run the reclass-generate-class-includes script to include them into the ./infra/maas_machines/machine_list.yml file. This file manages the class includes for all of the individual machine yaml files.

   ```
   ./reclass-generate-class-includes;
   ```

9. In the default ./infra/maas_machines.yml file appended the following to the very top of it:

   ```
   classes:
   - cluster.dvtcos.infra.maas_machines.machine_list
   ```

  
