dhcp_server
===========

This role installs and configures a DHCP server.

Requirements
------------

This role requires Ansible 1.4 or higher and platform requirements are listed in the metadata file.

Ubuntu AppArmor
---------------
Since Ubuntu 14.04, AppArmor is configured to not allow dhcpd to access files outside a certain list of paths.
This prevents Ansible from running the check command on the template. The check is used to validate the correctness of the config file generated.

To prevent this, you can either disable AppArmor, manually configure it in such a way that it allows access to `/root/.ansible/tmp` for dhcpd or you can let this role do that for you:

If you specify the `configure_apparmor: true` variable for your host. This role will overwrite the `/etc/apparmor.d/local/usr.bin.dhcpd` file and specifically allow read-only access to `/root/.ansible/tmp`. It will first check if this file exists, if it does not, it will not do anything.

Difference between global and subnet interface options
-------------------------------------------------------
Global dhcp_interfaces option makes listen on defined interfaces all subnets. Interface per subnet definition allows listen as much subnets as you want.
Global dhcp_interfaces option does not work on systemd distros (ArchLinux, CentOS 7, Fedora), listen by default on interface with declared subnet. You cat rewrite systemd service, but is dirty. Instead this, describe interfaces in configuration. Is modern and properly.

Role Variables
--------------

The variables that can be passed to this role and a brief description about
them are as follows. These are all based on the configuration variables of the
DHCP server configuration.

    # AppArmor configuration - important for Ubuntu 14.04
    configure_apparmor: true

    # Basic configuration information
    dhcp_interfaces: eth0
    dhcp_common_domain: example.org
    dhcp_common_nameservers: ns1.example.org, ns2.example.org
    dhcp_common_default_lease_time: 600
    dhcp_common_max_lease_time: 7200
    dhcp_common_ddns_update_style: none
    dhcp_common_authoritative: true
    dhcp_common_log_facility: local7
    dhcp_common_options:
    - opt66 code 66 = string
    dhcp_common_parameters:
    - filename "pxelinux.0"

    # Subnet configuration
    dhcp_subnets:
    # Required variables example
    - base: 192.168.1.0
      netmask: 255.255.255.0
    # Full list of possibilities
    - base: 192.168.10.0
      netmask: 255.255.255.0
      interface: vlan100
      range_start: 192.168.10.150
      range_end: 192.168.10.200
      routers: 192.168.10.1
      broadcast_address: 192.168.10.255
      domain_nameservers: 192.168.10.1, 192.168.10.2
      domain_name: example.org
      ntp_servers: pool.ntp.org
      default_lease_time: 3600
      max_lease_time: 7200
      pools:
      - range_start: 192.168.100.10
        range_end: 192.168.100.20
        rule: 'allow members of "foo"'
        parameters:
        - filename "pxelinux.0"
      - range_start: 192.168.110.10
        range_end: 192.168.110.20
        rule: 'deny members of "foo"'
      parameters:
      - filename "pxelinux.0"

    # Fixed lease configuration
    dhcp_hosts:
    - name: local-server
      mac_address: "00:11:22:33:44:55"
      fixed_address: 192.168.10.10
      default_lease_time: 43200
      max_lease_time: 86400
      parameters:
      - filename "pxelinux.0"

    # Class configuration
    dhcp_classes:
    - name: foo
      rule: 'match if substring (option vendor-class-identifier, 0, 4) = "SUNW"'
    - name: CiscoSPA
      rule: 'match if (( substring (option vendor-class-identifier,0,13) = "Cisco SPA504G" ) or
             ( substring (option vendor-class-identifier,0,12) = "Cisco SPA303" ))'
      options:
      - opt: 'opt66 "http://distrib.local/cisco.php?mac=$MAU"'
      - opt: 'time-offset 21600'

    # Shared network configurations
    dhcp_shared_networks:
    - name: shared-net
      interface: vlan100
      subnets:
      - base: 192.168.100.0
        netmask: 255.255.255.0
        routers: 192.168.10.1
      parameters:
      - filename "pxelinux.0"
      pools:
      - range_start: 192.168.100.10
        range_end: 192.168.100.20
        rule: 'allow members of "foo"'
        parameters:
        - filename "pxelinux.0"
      - range_start: 192.168.110.10
        range_end: 192.168.110.20
        rule: 'deny members of "foo"'

    # Custom if else clause
      dhcp_ifelse:
      - condition: 'exists user-class and option user-class = "iPXE"'
        val: 'filename "http://my.web.server/real_boot_script.php";'
        else:
          - val: 'filename "pxeboot.0";'
          - val: 'filename "pxeboot.1";'

Examples
========

1) Install DHCP server on interface eth0 with one simple subnet:

    - hosts: all
      roles:
      - role: dhcp_server
        dhcp_interfaces: eth0
        dhcp_common_domain: example.org
        dhcp_common_nameservers: ns1.example.org, ns2.example.org
        dhcp_common_default_lease_time: 600
        dhcp_common_max_lease_time: 7200
        dhcp_common_ddns_update_style: none
        dhcp_common_authoritative: true
        dhcp_common_log_facility: local7
        dhcp_subnets:
        - base: 192.168.10.0
          netmask: 255.255.255.0
          range_start: 192.168.10.150
          range_end: 192.168.10.200
          routers: 192.168.10.1


2) Install DHCP server with subnet per interface:

    - hosts: all
      roles:
      - role: dhcp_server
        dhcp_common_domain: example.org
        dhcp_common_nameservers: ns1.example.org, ns2.example.org
        dhcp_common_default_lease_time: 600
        dhcp_common_max_lease_time: 7200
        dhcp_common_ddns_update_style: none
        dhcp_common_authoritative: true
        dhcp_common_log_facility: local7
        dhcp_subnets:
        - base: 192.168.10.0
          netmask: 255.255.255.0
          interface: vlan10
          range_start: 192.168.10.150
          range_end: 192.168.10.200
          routers: 192.168.10.1
        - base: 192.168.20.0
          netmask: 255.255.255.0
          interface: vlan20
          range_start: 192.168.20.150
          range_end: 192.168.20.200
          routers: 192.168.20.1


3) Install DHCP server with one subnet on interface vlan10 and with shared network on interface vlan20

    - hosts: all
      roles:
      - role: dhcp_server
        dhcp_common_default_lease_time: 600
        dhcp_common_max_lease_time: 7200
        dhcp_common_ddns_update_style: none
        dhcp_common_authoritative: true
        dhcp_common_log_facility: local7
        dhcp_subnets:
        - base: 192.168.10.0
          netmask: 255.255.255.0
          interface: vlan10
          domain_nameserver: 192.168.10.1
          domain_name: example.local
          range_start: 192.168.10.150
          range_end: 192.168.10.200
          routers: 192.168.10.1
        dhcp_shared_networks:
        - name: sharednet
          interface: vlan20
          subnets:
          - base: 10.7.0.0
            netmask: 255.255.255.0
            routers: 10.7.0.1
            domain_nameserver: 10.7.0.1
            domain_name: example.public0
            ntp_servers: 10.7.0.1
            pools:
            - range_start: 10.7.0.2
              range_end: 10.7.0.254
          - base: 10.8.0.0
            netmask: 255.255.255.0
            routers: 10.8.0.1
            domain_nameserver: 10.8.0.1
            domain_name: example.public1
            ntp_servers: 10.8.0.1
            pools:
            - range_start: 10.8.0.2
              range_end: 10.8.0.254


Dependencies
------------

None

License
-------

BSD

Author Information
------------------

Philippe Dellaert
