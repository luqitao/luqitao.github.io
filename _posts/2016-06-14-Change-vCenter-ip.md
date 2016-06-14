---
layout: post
title: how-to-change-vcenter-server-appliance-ip-from-command-line
tags: vCenter
---

通过命令行修改vcenter server ip的方法
---

When deploying the VMware vCenter Server Appliance (VCSA) it will default look for a DHCP address. When there is no DHCP server available the following error is displayed:

NO NETWORKING DETECTED.

![no_networking_detected](images/no_networking_detected.png)

it is possible to manually configure a static IP address by using the command line. Here are the steps:

- Open a console session of the VCSA 
- Login as: root
- Default password is: vmware
- Execute the following command:

```
$ /opt/vmware/share/vami/vami_config_net
```

After executing the command, a menu is displayed. Within the menu It is possible to change the IP address, hostname, DNS, Default gateway and proxy server.


![vami_config_net](images/vami_config_net.png)

After allocate a static IP Address to the VCSA the post configuration can be done by using the following URL: 
[https://static-ip-address:5480](https://static-ip-address:5480)

[原文链接](http://www.ivobeerens.nl/2013/01/14/allocate-a-static-ip-address-to-the-vmware-vcenter-server-appliance-vcsa/)