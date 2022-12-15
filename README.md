# The purpose of this repository is to explain how to install InfluxDB, Telegraf and Grafana to collect, store and visualize gRPC Streaming Telemetry Data from Cisco WLCs.

## This installation has been tested and validated with Ubuntu Server 22.04

show version of Ubuntu:
```bash
root@collector1:~# lsb_release -a && ip r

Output:
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04.1 LTS
Release:	22.04
Codename:	jammy
default via 10.93.178.129 dev ens160 proto static
```
Before installing the 3 software's, update your package source list to the latest version of the repository and upgrade your Ubuntu OS:
```bash
root@collector1:# apt update
root@collector1:# apt list --upgradable
root@collector1:# apt upgrade
```
If you are behind a proxy, make sure you add the following lines on your .bashrc file. This is to avoid having issues with the connectivity of the collectors with a centralized InfluxDB:
```bash
root@collector1:# vi ~/.bashrc
printf -v no_proxy '%s,' 10.93.178.{140..160};
export no_proxy="${no_proxy%,}";
 ```
It can also be done by adding the line bellow on: /etc/environment for all your collectors that would like to be outside your proxy settings:
```bash
root@collector1:# vi /etc/environment
Export NO_PROXY="localhost,127.0.0.1,.localdomain,.internal,10.93.178.140,10.93.178.141,10.93.178.142,10.93.178.142,10.93.178.143,10.93.178.144, .cisco.com"
```



