% Troubleshooting Network Services (on EL7)
% Enterprise Linux
% HOGENT applied computer science

# Preparation

## Before we begin

Set up the test environment

```console
$ cd elnx-syllabus/demo
$ vagrant up web db
[...]
```

# Introduction

## Agenda

- Bottom-up approach
- Network access (Link) layer
- Internet layer
- Transport
- Application Layer
- SELinux

**Interrupt me if you have remarks/questions!**

## Terminology

- *host system* = the physical machine running virtualisation software (my laptop)
- *network host* = any machine with an IP address

## Case: web + db server

Two VirtualBox VMs, set up with Vagrant

| Host  | IP            | Service              |
| :---  | :---          | :---                 |
| `web` | 192.168.56.72 | http, https (Apache) |
| `db`  | 192.168.56.73 | mysql (MariaDB)      |

- On `web`, a PHP app runs a query on the `db`
- `db` is set up correctly, `web` is not

## Objective

![The PHP application](img/result.png)

## Test the database server

```shell
$ ./query_db.sh 
+ mysql --host=192.168.56.73 --user=demo_user \
+   --password=ArfovWap_OwkUfeaf4 demo \
+   '--execute=SELECT * FROM demo_tbl;'
+----+-------------------+
| id | name              |
+----+-------------------+
| 1  | Tuxedo T. Penguin |
| 2  | Bobby Tables      |
+----+-------------------+
+ set +x
```

## Use a bottom-up approach

TCP/IP protocol stack

| Layer          | Protocols                | Keywords              |
| :---           | :---                     | :---                  |
| Application    | HTTP, DNS, SMB, FTP, ... |                       |
| Transport      | TCP, UDP                 | sockets, port numbers |
| Internet       | IP, ICMP                 | routing, IP address   |
| Network access | Ethernet                 | switch, MAC address   |
| Physical       |                          | cables                |

# Network Access Layer

## Network Access Layer

- bare metal:
    - test the cable(s)
    - check switch/NIC LEDs
- VM (e.g. VirtualBox):
    - check virtual network adapter type & settings
- `ip link`

# Internet Layer

## Checklist: Internet Layer

1. *Local* network configuration
2. Routing within the *LAN*

**Know the expected values!**

## Checklist: Internet Layer

Checking *Local network configuration:*

1. IP address: `ip a`
2. Default gateway: `ip r`
3. DNS service: `/etc/resolv.conf`

## Local configuration: `ip address`

- IP address?
- In correct subnet?
- DHCP or fixed IP?
- Check configuration: `/etc/sysconfig/network-scripts/ifcfg-*`

---

Example: DHCP

```console
[vagrant@db ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s3 
TYPE=Ethernet
BOOTPROTO=dhcp
NAME=enp0s3
DEVICE=enp0s3
ONBOOT=yes
[...]
```

---

Example: Static IP

```console
$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.56.73
NETMASK=255.255.255.0
DEVICE=enp0s8
[...]
```

## Common causes (DHCP)

- No IP
    - DHCP unreachable
    - DHCP won't give an IP
- 169.254.x.x
    - No DHCP offer, "link-local" address
- Unexpected subnet
    - Bad config (fixed IP set?)

Watch the logs: `sudo journalctl -f`

## Common causes (Fixed IP)

- Unexpected subnet
    - Check config
- Correct IP, "network unreachable"
    - Check network mask

## Local configuration: `ip route`

- Default GW present?
- In correct subnet?
- Check network configuration

## DNS server: `/etc/resolv.conf`

- `nameserver` option present?
- Expected IP?

## Checklist: Internet Layer

Checking routing within the *LAN*:

- Ping between hosts
- Ping default GW/DNS
- Query DNS (`dig`, `nslookup`, `getent`)

## LAN connectivity: `ping`

- host -> VM: `ping 192.168.56.72`
- VM -> host: `ping 192.168.56.1`
- VM -> GW: `ping 10.0.2.2`
- VM -> DNS: `ping 10.0.2.3`

Remark: some routers **block** ICMP!

## LAN connectivity: DNS

- `dig icanhazip.com`
- `nslookup icanhazip.com`
- `getent ahosts icanhazip.com`

## LAN connectivity

Next step: routing beyond GW

# Transport Layer

## Checklist: Transport Layer

1. Service running? `sudo systemctl status SERVICE`
2. Correct port/inteface? `sudo ss -tulpn`
3. Firewall settings: `sudo firewall-cmd --list-all`

## Is the service running?

`systemctl status httpd.service`

- `active (running)` vs. `inactive (dead)`
    - `systemctl start httpd`
    - Fail? See below (Application layer)
- Start at boot: `enabled` vs. `disabled`
    - `systemctl enable httpd`

## Correct ports/interfaces?

- Use `ss` (not `netstat`)
    - TCP service: `sudo ss -tlnp`
    - UDP service: `sudo ss -ulnp`
- Correct port number?
    - See `/etc/services`
- Correct interface?
    - Only loopback?

## Firewall settings

`sudo firewall-cmd --list-all`

- Is the service or port listed?
- Use `--add-service` if possible
    - Supported: `--get-services`
- Don't use both `--add-service` and `--add-port`
- Add `--permanent`
- `--reload` firewall rules

---

```console
$ sudo firewall-cmd --add-service=http --permanent
$ sudo firewall-cmd --add-service=https --permanent
$ sudo firewall-cmd --reload
```

# Application Layer

## Checklist: Application Layer

- Check the *logs*: `journalctl`
- Validate config file *syntax*
- Use (command line) *client* tools
    - e.g. `curl`, `smbclient` (Samba), `dig` (DNS), etc.
- Other checks are application dependent
    - Read the reference manuals!

## Check the log files

- Either `journalctl`: `journalctl -f -u httpd.service`
- Or `/var/log/`:
    - `tail -f /var/log/httpd/error_log`

## Check config file syntax

- Application dependent, for Apache: `apachectl configtest`

## Read the fine manual!

- [RedHat Manuals](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/):
    - System Administrator's Guide
    - Networking guide
    - SELinux guide
- Reference manuals, e.g.:
    - <https://httpd.apache.org/docs/2.4/configuring.html>
- Man pages
    - smb.conf(5), dhcpd.conf(5), named.conf(5), ...

# SELinux troubleshooting

## SELinux

- SELinux is Mandatory Access Control in the Linux kernel
- Settings:
    - Booleans: `getsebool`, `setsebool`
    - Contexts, labels: `ls -Z`, `chcon`, `restorecon`
    - Policy modules: `sepolicy`

## Do not disable SELinux

<https://stopdisablingselinux.com/>

## Check file context

- Is the file context as expected?
    - `ls -Z /var/www/html`
- Set file context to default value
    - `sudo restorecon -R /var/www/`
- Set file context to specified value
    - `sudo chcon -t httpd_sys_content_t test.php`

## Check booleans

`getsebool -a | grep http`

- Know the relevant booleans! (RedHat manuals)
- Enable boolean:
    - `sudo setsebool -P httpd_can_network_connect_db on`

## Creating a policy

Let's try to set `DocumentRoot "/vagrant/www"`

```
$ sudo vi /etc/httpd/conf/httpd.conf
$ ls -Z /vagrant/www/
-rw-rw-r--. vagrant vagrant system_u:object_r:vmblock_t:s0   test.php
$ sudo chcon -R -t httpd_sys_content_t /vagrant/www/
chcon: failed to change context of ‘test.php’ to ‘system_u:object_r:httpd_sys_content_t:s0’: Operation not supported
chcon: failed to change context of ‘/vagrant/www/’ to ‘system_u:object_r:httpd_sys_content_t:s0’: Operation not supported
```

## Creating a policy

Instead of setting the files to the expected context, allow httpd to access files with `vmblock_t` context

1. Allow Apache to run in "permissive" mode:

    ```
    $ sudo semanage permissive -a httpd_t
    ```

2. Generate "Type Enforcement" file (.te)

    ```
    $ sudo audit2allow -a -m httpd-vboxsf > httpd-vboxsf.te
    ```

3. If necessary, edit the policy

    ```
    $ sudo vi httpd-vboxsf.te
    ```

---

1. Convert to policy module (.pp)

    ```
    $ checkmodule -M -m -o httpd-vboxsf.mod httpd-vboxsf.te
    $ semodule_package -o httpd-vboxsf.pp -m httpd-vboxsf.mod
    ```

5. Install module

    ```
    $ sudo semodule -i httpd-vboxsf.pp
    ```

6. Remove permissive domain exception

    ```
    $ sudo semanage permissive -d httpd_t
    ```

Tip: automate this!

# General guidelines

## Back up config files before changing

## Be systematic, bottom-up

## Be thorough, don't skip steps

## Do not assume: test

## Know your environment

## Know your log files

![Credit: @KrisBuytaert](img/reading-errorlog-files-small.jpg)

## Read The F*** Error Message!

## Open logs in separate terminal

## Small steps

## Validate the syntax of config files

## Reload service after config change

## Verify each change

## Keep a cheat sheet/checklist

E.g. <https://github.com/bertvv/cheat-sheets>

## Use a configuration management system

## Automate tests

E.g. <https://github.com/HoGentTIN/elnx-sme/blob/master/test/pu004/lamp.bats>

## Never ping Google!

# That's it!
