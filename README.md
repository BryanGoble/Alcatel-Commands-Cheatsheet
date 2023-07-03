# Alcatel Commands Cheatsheet

<!-- ![GitHub Repo stars](https://img.shields.io/github/stars/BryanGoble/alcatel-command-cheatsheet) -->
<!-- ![GitHub Watchers](https://img.shields.io/github/watchers/BryanGoble/alcatel-command-cheatsheet) -->
<!-- ![GitHub Forks](https://img.shields.io/github/forks/BryanGoble/alcatel-command-cheatsheet) -->

This repository is a continuation of what I consider to be the original official Alcatel cheatsheet found at the [Website of Latouche](http://www.latouche.info/admin/user_guides/omniswitch.html) by Vincent Touchard updated in 2015. Latouche has saved my butt on so many occasions and throughout the past few years since I began maintaining and operating a good number of Omniswitch 6850, 6850E, and 9700 switches at my workplace. I decided to build this repository to expand on the information provided by Latouche and some additional command sets that I've discovered along the way molding this into the master cheatsheet available for all Network Admins still using Omniswitch products.

<div id="back-to-top"></div>

## --- Table of Contents ---
|                                                                 Content | Sub-Content                               |
| ----------------------------------------------------------------------: | :---------------------------------------- |
| **[Config File Management](#configuration-config-file-management-📁)**: | **[Save/Reload](#savereload-config)**     |
| **[Interfaces](#interfaces-🔌)**                                        |                                           |
| **[Configure VLANs](#configure-vlans)**:                                | **[Port Association](#port-association)** |
| **[Link Aggregation](#link-aggregation-lag-⛓️)**:                       | **[Dynamic Lag](#dynamic-lag-lacp)**      |
|                                                                         | **[Static Lag](#static-lag)**             |
| **[Network Time Protocol](#network-time-protocol-ntp-⏰)**              |                                           |
| **[Spanning Tree Protocol](#spanning-tree-protocol-stp-🌲)**            |                                           |
| **[Domain Name System](#domain-name-system-dns)**                       |                                           |
| **[DHCP Relay](#dynamic-host-configuration-protocol-dhcp-relay)**       |                                           |
| **[Services](#services)**                                               |                                           |
| **[ARP](#address-resolution-protocol-arp)**                             |                                           |
| **[SNMP](#simple-network-management-protocol-snmp)**                    |                                           |
| **[Port Mirroring](#port-mirroring-🪞)**                               |                                            |
| **[Power Over Ethernet](#power-over-ethernet-poe-💡)**:                | **[Status](#status)**                      |
|                                                                         | **[Power Limits](#power-limits)**         |
| **[Hardware](#hardware-🖥️)**:                                          | **[Stacking](#stacking)**                  |
|                                                                         | **[Hardware Health](#hardware-health)**   |
| **[System](#system)**                                                   |                                           |
| **[Logs](#logs-🗂️)**                                                   |                                            |
| **[AAA](#authentication-authorization--accounting-aaa)**                |                                           |
| **[QoS & ACL](#quality-of-service-qos--access-control-list-acl)**       |                                           |

---

## Configuration (Config) File Management :file_folder:
Alcatel Omniswitches operate in either of two modes: **working** or **certified**.
- Run `show running-directory` to know which mode the switch is currently in.
    - In working mode, the configuration can be modified and _should_ be used for all config changes. According to Alcatel, it's not possible to make changes while in certified mode, but there's an easy work around below.
- During the bootup process, if working and certified configuration files are different, the switch will boot into certified mode.
- Configuration files are stored in `working/boot.cfg` and `certified/boot.cfg`.
    - These can be directly edited with `vi`.
>
When modifying the config, it can be useful to reload the switch into certified mode if a configuration error occurs.
- It's possible to set a reload timer in case you lose control of the switch during modification, use `reload in <n>` where `<n>` is the number of minutes to wait before reloading.
- To show when the switch will reboot, use `show reload`.
- To cancel the reload, use `reload cancel`.

### Save/Reload Config:
- Save running config => working: `write memory`
- Save working config => certified: `copy working certified`
- Save working config => certified (multi slot): `copy working certified flash-synchro`
- Save running config while in certified mode: `configuration snapshot all <file name>` then move file to `working/boot.cfg`
- Reboot into working at startup without rollback: `reload working no rollback-timeout`
- View running config: `show configuration snapshot <all|vlan|ip|...>` or `write terminal`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Interfaces :electric_plug:
- Global status: `show interfaces status`
- Interface details (status, MAC, speed, duplex, errors, ...): `show interfaces [port|status|<slot>/<port>|...]`
- Disable interface: `interface <slot>/<port> admin down`
- Summary of all interface errors: `show interfaces counters errors`
- Clear counters on specified interface(s): `interfaces <slot>/<port>[<port1-port2>] no l2 statistics`
- Modify interface details: `interface <slot>/<port> [speed <10, 100, 1000>|duplex <half, full>|autoneg <state>|flood rate <rate>]`
>
Example - Switch from autonegotiation to 100FD:
1. `interface <slot>/<port> autoneg off`
2. `interface <slot>/<port> speed 100 duplex full`
- If forced into 100FD while autoneg is on, the port will in `down` status

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Configure VLANs
- Create a layer 2 VLAN: `vlan <vlan_number> enable name "vlan name"`
- Remove a VLAN: `no vlan <vlan_number>`
- List all available VLANs: `show vlan`
- List details of a specific VLAN: `show vlan <vlan_number>`
>
<!-- Expand upon microcode -->
Depending on the microcode version, `show microcode`, a layer 3 VLAN can be created with:
1. `ip interface "interface name" vlan <vlan_number> address <address> mask <netmask>`
2. `vlan router "interface name" vlan <vlan_number> address <address> mask <netmask>`
>
and destroyed with:
1. `no ip interface "inteface name"`
2. `no vlan router "interface name"`
>
### Port association:
- Associate a port a specific vlan: `vlan <vlan_number> port default <slot>/<port>`
- List all ports and assigned vlans: `show vlan port`
- List all ports of a specific vlan: `show vlan <vlan_number> port`
- Show assigned vlan of specific port: `show vlan port <slot>/<port>`
* 802.1Q Tagging:
    * Tag a port: `vlan <vlan_number> 802.1Q <slot>/<port> [<"comment">]`
    * Remove a tag: `vlan <vlan_number> no 802.1Q <slot>/<port>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Link Aggregation (LAG) :chains:
### Dynamic LAG (LACP)
1. `lacp linkagg <id> size <size> admin state enable`
2. `lacp linkagg <id> actor admin key <key>`
3. `lacp agg <slot>/<port> actor admin key <key>`
>
### Static LAG
1. `static linkagg <id> size <size> admin state enable`
2. `static linkagg <id> name <name>`
3. `static agg <slot>/<port> agg num <id>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Network Time Protocol (NTP) :alarm_clock:
- Set NTP server address: `ntp server <server_ip>`
    - Do not use hostname, even if DNS is configured.
- Activate NTP connection: `ntp client enable`
- Get NTP details:
    - `show ntp client` - Displays NTP status (on/off) & last update time
    - `show ntp server-list` - Displays the list of configured NTP servers and which is synchronized

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Spanning Tree Protocol (STP) :evergreen_tree:
STP operates in one of two modes: **flat** and **1x1**. In flat mode, there is one instance for the whole switch. However, in 1x1 mode, there is one instance per VLAN (similar to Per VLAN Spanning Tree (PVST) with Cisco or VLAN STP with Juniper)
- 1x1 mode is recommended if there is no need for Multiple STP (MSTP)
>
- Modify STP mode: `bridge mode [flat|1x1]`
- View STP configuration: `show spantree`
- Activate/Deactivate STP on specified vlans/ports:
    1. `vlan <vlan_number> stp [enable|disable]`
    2. `bridge <vlan_number> <slot>/<port> [enable|disable]`
- Modify STP Algorithm: `bridge protocol [802.1D|STP|RTSP]`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Domain Name System (DNS)
- Specify name servers: `ip name-server <ip_address1> <ip_address2>`
- Specify doman name: `ip domain-name <domain_name>`
- Activate DNS client: `ip domain-lookup`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Dynamic Host Configuration Protocol (DHCP) Relay
- Enable DHCP relay service: `ip service udp-relay`
- Restrict DHCP relay only for vlans: `ip helper per-vlan only`
- Configure DHCP relay for specified vlan: `ip helper address <dhcp_server> vlan <vlan_number>`
- Enable DHCP relay: `ip udp relay BOOTP`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Services
- Activate/Deactivate services: `[no] ip service [ftp|ssh|telnet|http|secure-http|udp-relay|snmp|all]`
    - You can enable/disable multiple services at the same time by separating service names with a **space**.
        - Example - `ip service telnet ftp snmp`
    - It's also possible to enable/disable services by their common port numbers: `[no] ip service [21|22|23|80|443|...]`
- View active services: `show ip service`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Address Resolution Protocol (ARP)
- Display ARP table: `show arp`
- Display MAC address table: `show mac-address-table`
- Add a static ARP entry: `arp <ip_address> <mac_address>`
- Remove a static ARP entry: `no arp <ip_address>`
- Clear dynamic ARP entries: `clear arp-table`
- Specify dynamic entry timeouts (default: 300 seconds): `mac-address-table aging-time <seconds> [vlan <vlan_number>]`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Simple Network Management Protocol (SNMP)
To enable SNMP:
1. `ip service snmp` - Enable SNMP service
2. `user <user_name> password <new_password> read-only all no auth` - Create a user/give SNMP permissions
3. `aaa authentication snmp "local"` - Enable authentication
4. `snmp security no security` - Set SNMP security
5. `snmp community map "public" user <user_name> on` - Associate community string with user
6. `snmp station <server_ip> [<port>] <"user_name"> [v1|v2|v2c|v3] enable` - Configure SNMP trap server
>
- Enable/Disable snmp trap authentication: `snmp authentication trap [enable|disable]`
- Filter traps sent by the switch: `snmp trap filter <server_ip> <filter_code>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Port Mirroring :mirror:
<!-- Explain this -->
Port mirroring works 12 ports by 12 ports. It is possible to configure multiple sources for one session and thus see the traffic of multiple ports in one output.
- Display mirroring status: `show port mirroring status`
- Enable port mirroring: `port mirroring <session> source <slot>/<port> destination <slot>/<port> enable`
- Disable port mirroring: `no port mirroring <session>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Power Over Ethernet (POE) :bulb:
By default, POE is disabled on all ports.
- Enable POE on the entire slot/switch: `lanpower start <slot>`
- Enable POE on a specific port: `lanpower start <slot>/<port>`
- Disable POE: `lanpower stop [<slot>/<port>|<slot>]`
>
### Status
- Show POE configuration: `show lanpower <slot>`
>
### Power Limits
- Limit the power available on a specic port: `lanpower <slot>/<port> power <milliwatts>`
- Limit the power available on a slot: `lanpower <slot> maxpower <watts>`
>
There is a known issue with Alcatel switches proving unstable when providing POE to too many devices at a time with an underpowered PSU. A common rule of thumb is factoring 15W of power per device.
- (PSU Wattage - 100 Watts reserved for switch) / 15W = # of POE devices

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Hardware :desktop_computer:
### Stacking
- When switches are operating in a stack, one switch will be primary, one will be secondary, and all others will be set to idle.
- If the primary is down, the secondary becomes primary and the first idle becomes secondary.
- Show info about the stack: `show stack topology`
### Hardware health
- Show info about the current switch hardware: `show chassis`
- Monitor health of the switch: `show health all [cpu|memory]`
- Monitor health of specific interface: `show health <slot>/<port>`
- Reset health statistics: `health statistics reset`
- Show Control Management Module (CMM) info: `show cmm`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## System
- Show system details (uptime, date, name, location): `show system`
- Modify system details:
    1. `system name <"name">`
    2. `system contact <"contact">`
    3. `system location <"location">`
- Show session parameters: `show session config`
- Modify default session prompt from **->** to **sw1->**: `session prompt default "sw1->"`
- When a command outputs too many lines on the screen, use `more` to view outputs page by page.
    - `more size <size>` sets the number of lines shown
    - `no more` cancels this mode
- Change the timeout of telnet/ssh sessions: `session timeout cli <seconds>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Logs :card_index_dividers:
- Show logging configuration: `show swlog`
- Display all switch logs: `show log swlog`
    - Show logs since specified time: `show log swlog timestamp <month/day/year> <hour:minute>`
- Clear all logs: `swlog clear`
- Enable syslog: `swlog output socket <syslog_server_ip>`

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Authentication, Authorization, & Accounting (AAA)
Authentication can be local or configured with RADIUS
- Set authentication: `aaa authentication default "local", aaa authentication [console|ssh|ftp|802.1X|vlan|...] "local"`

Example 802.1X Configuration:
```
aaa radius-server "radius_srv1" host <ip_address> key <auth_key> retransmit 3 timeout 2 auth-port 1812 acct-port 1813
aaa radius-server "radius_srv2" host <ip_address> key <auth_key> retransmit 3 timeout 2 auth-port 1812 acct-port 1813

# Use the radius for vlan assignment
aaa authentication vlan single-mode "radius_srv1" "radius_srv2"

# Use the internal database for authent to the local services
aaa authentication default "local"
aaa authentication console "local"
aaa authentication ftp "local"
aaa authentication snmp "local"

# 801.1X authentication servers
aaa authentication 802.1x radius_srv1 radius_srv2

# MAC base authentication servers (used for devices that can't do 802.1X like IP-Phones)
aaa authentication mac radius_srv1 radius_srv2

# AVLAN:
#   Authentication portal in the switch. By default, last IP of the subnet.
avlan auth-ip <vlan-ID> <IP address, in same VLAN, different of switch IP address>

# VLAN definition
vlan 5 enable name "VoIP"
vlan 10 enable name "Data"
vlan 10 authentication enable

# Configuration of interface 1/3
vlan 10 port default 1/3
# enable dynamic vlan assignemt
vlan port mobile 1/3
# enable 802.1X 
vlan port 1/3 802.1x enable

# 802.1X
#   Direction both => control on inbound + outbound traffic
#   Port-control auto => port initially in unauthorized state, and put in "authorized mode" automatically by the switch upon the exchanged between the switch and the end station
#   Quiet-period 60 => reject the 802.1X authentications during 60s after an authentication failure
#   Server-timeout 30 => superseded by the aaa radius-server ... timeout
#   Re-authperiod 3600 => 3600s=1h before re-authent is required
#   No reauthentication => disables the reauthent
802.1x 1/3 direction both port-control auto quiet-period 60 tx-period 30 supp-timeout 30 server-timeout 30 max-req 2 re-authperiod 3600 no reauthentication

# Length of a captive portal session
802.1x 1/3 captive-portal session-limit 12 retry-count 3

# Poll the end device 2 times before stating it is not 802.1X compliant
802.1x 1/3 supp-polling retry 2

# if authentication is successful but returns no VLAN ID ("pass"), use default vlan for the supplicant else ("fail"), block the port
802.1x 1/3 supplicant policy authentication pass group-mobility default-vlan fail block

# Idem for non supplicant (not 802.1X) devices - authentication by MAC address with a Radius
802.1x 1/3 non-supplicant policy authentication pass group-mobility block fail block

# Used by supplicant and non supplicant when "captive-portal" is used in the "802.1x supplicant policy" or "802.1x non-supplicant policy"
802.1x 1/3 captive-portal policy authentication pass default-vlan fail block
```

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## Quality of Service (QoS) & Access Control List (ACL)
In AOS, ACL and QoS are configured in the same **QoS** section
- Apply QoS changes: `qos apply`
- Disable QoS: `qos disable`

By default, QoS is not trusted in access ports and all of its tags are set to **0**. However, it is trusted on trunked ports.
- To trust QoS everywhere: `qos trust ports`
- To trust on a specific port: `qos port <slot>/<port> trusted`

QoS & ACL rules are defined by a combination of elements:
 - Policy network: define subnets
    - `policy network group <group_name> <subnet1> mask <mask1> <subnet2> mask <mask2> ...`
 - Policy condition: define conditions (from subnet1 to subnet2, ...)
    - `policy condition <condition_name> source network group <group_name1> destination group <group_name2>`
 - Policy action: define actions (permit, deny, ...)
    - `policy action <action_name> disposition <action>`
 - Policy rule: apply action to condition (if X then Y)
    - `policy rule <rule_name> [disable] [precedence <precedence>] condition <condition_name> action <action_name>` - Where precedence is the order in which rules can be applied

Example QoS & ACL Policy Configuration:
```
policy network group VoIP 192.168.1.0 mask 255.255.255.0 192.168.11.0 mask 255.255.254.0
policy network group Data 172.16.0.0 mask 255.255.255.0

policy condition "VoIP-VoIP" source network group VoIP destination network group VoIP
policy condition "VoIP-Data"  source network group VoIP destination network group Data
policy condition "Data-Data" source network group Data destination network group Data
policy condition "Other" source ip any destination ip any

policy action Deny disposition deny
policy action Permit

policy rule "Allow VoIP-VoIP" precedence 200 condition "VoIP-VoIP" action Permit
policy rule "Allow VoIP-Data" disable precedence 200 condition "VoIP-Data" action Permit
policy rule "Allow Data-Data" precedence 200 condition "Data-Data" action Permit
policy rule "Deny Other" precedence 200 condition "Other" action Deny

qos port 1/2 trusted
qos port 1/3 trusted
qos apply
```

<div align="right">&#8673; <a href="#back-to-top" title="Table of Contents">Back to Top</a></div>

## References
1. [Website of Latouche](http://www.latouche.info/admin/user_guides/omniswitch.html)
1. [OmniSwitch Configuration Guide](https://support.alcadis.nl/Support_files/Alcatel-Lucent/OmniSwitch//End%20of%20Sale%20products/OS6800%20-%20EOL/Manuals/OS6800%20AOS%206.1.5/OS6800%20AOS%206.1.5%20Network%20Configuration%20Guide.pdf) - Applicable to Models 6800, 6850, and 9000
1. [OmniSwitch CLI Reference Guide](https://support.alcadis.nl/Support_files/Alcatel-Lucent/OmniSwitch//OS6900/Manuals/OS6900%20AOS%207.2.1%20R01/OS6900%20AOS%207.2.1%20R01%20CLI%20Reference%20Guide.pdf) - Applicable to Models 10K and 6900