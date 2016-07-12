Ansible Ubuntu (Debian) network interfaces role
===============================================

Supports Ubuntu 12.04, 14.04 and 16.04

This role installs required networking packages and allows you to create `/etc/network/interfaces`
file based on `yaml` syntax.

Supported networking modes:
---------------------------
* Basic static interfaces
* Bonded static interfaces - `bond mode 1` and `bond mode 4`
* VLANS on both basic and bonded interfaces
* Custom routes to any of the interfaces / vlans
* Reordering / renaming interfaces that use old naming scheme (eth*)

Support was based on my current needs so adding any of the additional networking options should be filed in an Issue.

NOTE
----
```
Using this role wrong could result in loss of network connectivity. Author of this playbook
is NOT IN ANY WAY responsible if you loose your connectivity. You have been warned.

On the other hand this network interfaces role has been battle tested on
hundreds of Ubuntu servers on both 12.04 and 14.04.
```

Starting from Ubuntu 14.04 Ubuntu introduced new interface naming scheme which relies on hardware to map interfaces (i.e. em1)
so 70-persistent-net.rules fix is not needed (and is not imported in this role if version is not 12.04). However if you
want to remap existing interfaces to some naming convention that you like you can easily do this by changing interface
name in examples below and using 70-persistent-net.rules. Enabling the old interface naming convention in Ubuntu 14.04
requires to override one more parameter `biosdevname` in `71-biosdevname.rules`. This is however not tested on
Ubuntu 16.04 so this fix is also not imported.

Configuration Examples
----------------------
###### Note
```
If you don't want to use 70-persistent-net.rules you can ommit `mac` parameter. If you ARE using 70-persistent-net.rules
you need to define all of your interfaces since missing interface can cause interface naming confilct and you will
probably loose your connectivity since interface will be renamed.
```

#### Basic interfaces

In oder to define basic static interfaces you will need to define your interfaces like this:

```yaml
networks:
    - { description: 'Public Interface', interface: 'eth0', mac: '41:f3:ea:6c:ae:61', ip: '109.30.30.70', netmask: '255.255.255.224', network: '109.30.30.64', gateway: '109.30.30.65'}
    - { description: 'Corp Interface', interface: 'eth1', mac: '02:1c:22:c7:61:5a', ip: '192.168.1.20', netmask: '255.255.255.224', network: '192.168.1.0'}
    - { description: 'Unconfigured Interface', interface: 'eth5', mac: 'e5:1a:13:b3:ce:82'}
    - { description: 'Unconfigured Interface', interface: 'eth6', mac: 'e5:1a:13:b3:ce:83'}
```
- `description`: is used to generate comment before interface configuration in both config and 70-persistent-net.rules
- `interface`: is the interface name you want to configure, you can verify interface names with `ifconfig`
- `mac`: optional parameter used when using 70-persistent-net.rules to ensure that each interface will always have the same
    interface name, fixed in 14.04 and onwards
- `ip`: ip address of the interface
- `netmask`: netmask of the interface in long notation (i.e. 255.255.255.0)
- `network`: network parameter of the interface
- `gateway`: optional gateway parameter of the network interface, this will determine default gateway

#### Bonded interfaces

In order to define bonded interfaces you will need to define your interfaces like this:
```yaml
networks:
    - { description: 'Bonded Interface', interface: 'eth3', mac: 'e5:1a:13:b3:ce:80'}
    - { description: 'Bonded Interface', interface: 'eth4', mac: 'e5:1a:13:b3:ce:81'}
    - { description: 'Bonded Interface', interface: 'eth5', mac: 'e5:1a:13:b3:ce:82'}
    - { description: 'Bonded Interface', interface: 'eth6', mac: 'e5:1a:13:b3:ce:83'}
    - { description: 'Bonded Public Interface', interface: 'bond1', mac: '95:eb:bb:22:1c:48', ip: '109.30.30.71', netmask: '255.255.255.224', network: '109.30.30.64', gateway: '109.30.30.65', slaves: ['eth3', 'eth4'], bond_mode: '1' }
    - { description: 'Bonded Private Interface', interface: 'bond0', mac: '25:b1:cd:01:02:fc', ip: '10.10.10.1', netmask: '255.255.0.0', network: '10.10.0.0', slaves: ['eth5', 'eth6'], bond_mode: '4' }
```
- Basic parameters as for basic interfaces
- `slaves`: array of interface names, they have to correspond to existing interfaces. Order of interfaces in array defines
    primary interface in `bond mode 1`
- `bond_mode`: currently supports `bond mode 1 (active-backup) and 4 (802.3ad)`.

#### Vlans

In order to define VLANs define your interfaces like this:

```yaml
networks:
    - { description: 'STB Interface', interface: 'eth2', mac: '02:1c:22:c7:61:5b', vlan_interface: 'yes'}

vlans:
    - { description: 'Public Vlan', interface: 'eth2.1070', ip: '172.1.0.1', netmask: '255.255.255.248'}
    - { description: 'Private Vlan', interface: 'eth2.1071', ip: '172.1.0.2', netmask: '255.255.255.248'}
    - { description: 'Custom Vlan', interface: 'eth2.1072', ip: '172.1.0.3', netmask: '255.255.255.248'}
```
- VLAN interface cannot have IP address associated with it, only with its VLAN interfaces
- `vlan_interfaces`: flag to determine if interface should have vlans defined
- `interface` in `vlans`: full name of VLAN interface `<ifname>.<vlan_nr>` (i.e. eth2.1070)


#### Custom routes

In order to define custom routes define your routes like this:
```yaml
custom_routes:
    - { dest_network: '192.168.0.0', dest_netmask: '255.255.0.0', gateway: '192.168.5.1', interface: 'eth1' }
    - { dest_network: '172.16.0.0', dest_netmask: '255.255.0.0', gateway: '192.168.6.1', interface: 'eth1' }
    - { dest_network: '10.10.0.0', dest_netmask: '255.255.0.0', gateway: '172.1.0.1', interface: 'eth2.1079' }
    - { dest_network: '10.20.0.0', dest_netmask: '255.252.0.0', gateway: '172.1.0.1', interface: 'eth2.1079' }
```
- Interface can be any of the previously defined interfaces including basic interfaces (eth1), bonded interfaces (bond0),
    and also vlan interfaces (eth2.1070)


#### All mixed together
```yaml
networks:
    - { description: 'Public Interface', interface: 'eth0', mac: '41:f3:ea:6c:ae:61', ip: '109.30.30.70', netmask: '255.255.255.224', network: '109.30.30.64', gateway: '109.30.30.65'}
    - { description: 'Corp Interface', interface: 'eth1', mac: '02:1c:22:c7:61:5a', ip: '192.168.1.20', netmask: '255.255.255.224', network: '192.168.1.0'}
    - { description: 'STB Interface', interface: 'eth2', mac: '02:1c:22:c7:61:5b', vlan_interface: 'yes'}
    - { description: 'Bonded Interface', interface: 'eth3', mac: 'e5:1a:13:b3:ce:80'}
    - { description: 'Bonded Interface', interface: 'eth4', mac: 'e5:1a:13:b3:ce:81'}
    - { description: 'Bonded Interface', interface: 'eth5', mac: 'e5:1a:13:b3:ce:82'}
    - { description: 'Bonded Interface', interface: 'eth6', mac: 'e5:1a:13:b3:ce:83'}
    - { description: 'Bonded Storage Interface', interface: 'bond1', mac: '95:eb:bb:22:1c:48', ip: '109.30.30.71', netmask: '255.255.255.224', network: '109.30.30.64', gateway: '109.30.30.65', slaves: ['eth3', 'eth4'], bond_mode: '1' }
    - { description: 'Bonded Backup Interface', interface: 'bond0', mac: '25:b1:cd:01:02:fc', ip: '10.10.10.1', netmask: '255.255.0.0', network: '10.10.0.0', slaves: ['eth5', 'eth6'], bond_mode: '1' }

vlans:
    - { description: 'Public Vlan', interface: 'eth2.1070', ip: '172.1.0.1', netmask: '255.255.255.248'}
    - { description: 'Private Vlan', interface: 'eth2.1071', ip: '172.1.0.2', netmask: '255.255.255.248'}
    - { description: 'Custom Vlan', interface: 'eth2.1072', ip: '172.1.0.3', netmask: '255.255.255.248'}

custom_routes:
    - { dest_network: '192.168.0.0', dest_netmask: '255.255.0.0', gateway: '192.168.5.1', interface: 'eth1' }
    - { dest_network: '172.16.0.0', dest_netmask: '255.255.0.0', gateway: '192.168.6.1', interface: 'eth1' }
    - { dest_network: '10.10.0.0', dest_netmask: '255.255.0.0', gateway: '172.1.0.1', interface: 'eth2.1079' }
    - { dest_network: '10.20.0.0', dest_netmask: '255.252.0.0', gateway: '172.1.0.1', interface: 'bond1' }
```

Contributions
-------------
All contributions are welcome and strongly encouraged.

Requests
--------
Please report an Issue if you want some feature added or you have an actual issue while using this role. I'll be happy
to help.