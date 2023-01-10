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
## Installing InfluxDB on Ubuntu Server:
```bash
root@collector1:# apt install influxdb
root@collector1:# systemctl enable --now influxdb   # Enable InfluxDB to start when the server is restarted
root@collector1:# systemctl is-enabled influxdb
root@collector1:# systemctl restart influxdb
root@collector1:# systemctl status influxdb
```
Configure influxdb:
```bash
root@collector1:# apt install influxdb-client
root@collector1:# influx
Connected to http://localhost:8086 version 1.6.7~rc0
InfluxDB shell version: 1.6.7~rc0
> show databases
> create database mdt_grpc_tls
> show databases
> use mdt_grpc_tls
> show measurements  # This will be empty because when you install the DB there is no measurements created.
```
In case you want to use a specific username/password for the connection to the database in a remote location (not local):
If you define a user/pass, these credentials will be used by Grafana and Telegraf to connect to the Database.
```bash
root@collector1:# influx
> create user admin with password 'C1sco12345' with all privileges 
> show users
```

## Installing Telegraf on Ubuntu:
```bash
root@collector1:# apt install telegraf
root@collector1:# telegraf version
root@collector1:# systemctl status telegraf
root@collector1:# systemctl enable --now telegraf   # Enable Telegraf to start when the server is restarted
root@collector1:# systemctl is-enabled telegraf
In case telegraf haven't started yet, then go ahead and start the process:
root@collector1:# systemctl start telegraf
```
If you defined a user/pass for influxdb then you need to define the credentials in your influxdb plugin on /etc/telegraf/telegraf.conf
```bash
root@collector1:# vi /etc/telegraf/telegraf.conf
```
Add the following information:
```bash
# Output Plugin InfluxDB 
[[outputs.influxdb]] 
  database = "telegraf" 
  urls = [ "http://127.0.0.1:8086" ] 
  username = "telegraf" password = "telegraf"   # In case you defined the user/pass in the DB.
```
To verify on which port telegraf and influxDB is running execute the following command:
```bash
root@collector1:# netstat -tulpn

Example: 
root@collector1:/etc/telegraf# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      769/systemd-resolve 
tcp        0      0 127.0.0.1:34691         0.0.0.0:*               LISTEN      813/containerd      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1290/sshd: /usr/sbi 
tcp        0      0 127.0.0.1:8088          0.0.0.0:*               LISTEN      799/influxd         
tcp6       0      0 :::8086                 :::*                    LISTEN      799/influxd         
tcp6       0      0 :::3000                 :::*                    LISTEN      34820/grafana-serve 
tcp6       0      0 :::57500                :::*                    LISTEN      37413/telegraf      
tcp6       0      0 :::22                   :::*                    LISTEN      1290/sshd: /usr/sbi 
udp        0      0 127.0.0.53:53           0.0.0.0:*                           769/systemd-resolve 
```
Sniff of the telegraf.conf config on /etc/telegraf with a local influxDB. Keep in mind that you need to copy the certs to the server inside the directory defined in telegraf.conf file. In this case certs are defined /etc/telegraf/mtls/
```bash
root@collector1:# more /etc/telegraf/telegraf.conf 
# Global Agent Configuration
[agent]
  hostname = "collector1.amazonaccountteam.com"
  flush_interval = "15s"
  interval = "15s"
  logfile = "/var/log/telegraf/telegraf.log"

# gRPC + TLS
[[inputs.cisco_telemetry_mdt]]
transport = "grpc"
service_address = ":57500"
tls_cert = "/etc/telegraf/mtls/collector1.amazonaccountteam.com.crt"
tls_key = "/etc/telegraf/mtls/collector1.amazonaccountteam.com.key"

# telegraf-grpc-mtls.conf
# Enable TLS client authentication and define allowed CA certificates (mTLS)
tls_allowed_cacerts = ["/etc/telegraf/mtls/wlc1.9840.amazonaccountteam.com-ca.crt"]

[[outputs.file]]
  files = ["/etc/telegraf/telegraf-grpc-mtls.log"]

# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "mdt_grpc_tls"
  urls = [ "http://127.0.0.1:8086" ]
  #username = "admin"
  #password = "C1sco12345"   # In case you defined the user/pass in the DB.


Now lets proceed to create the log files defined on telegraf.conf:
touch /etc/telegraf/telegraf-grpc-mtls.log
chmod go+wr /etc/telegraf/telegraf-grpc-mtls.log

touch /var/log/telegraf/telegraf.log
chmod go+wr /var/log/telegraf/telegraf.log
```
Sniff of the telegraf.conf config on /etc/telegraf with a remote influxDB and user/pass to connect:
```bash
root@collector1:# more telegraf.conf
# Global Agent Configuration
[agent]
  hostname = "collector1.amazonaccountteam.com"
  flush_interval = "15s"
  interval = "15s"
  logfile = "/var/log/telegraf/telegraf.log"

# gRPC + TLS
[[inputs.cisco_telemetry_mdt]]
transport = "grpc"
service_address = ":57500"
tls_cert = "/etc/telegraf/mtls/collector1.amazonaccountteam.com.crt"
tls_key = "/etc/telegraf/mtls/collector1.amazonaccountteam.com.key"

# telegraf-grpc-mtls.conf
# Enable TLS client authentication and define allowed CA certificates (mTLS)
tls_allowed_cacerts = ["/etc/telegraf/mtls/wlc1.9840.amazonaccountteam.com-ca.crt"]

[[outputs.file]]
  files = ["/etc/telegraf/telegraf-grpc-mtls.log"]

# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "mdt_grpc_tls"
  urls = [ "http://10.93.178.143:8086" ]
  username = "telegraf" password = "telegraf"   # In case you defined the user/pass in the DB.
```

## Installing Grafana on Ubuntu Server:
```bash
root@collector1:# apt install -y adduser libfontconfig1
root@collector1:# wget  https://dl.grafana.com/enterprise/release/grafana-enterprise_9.3.1_amd64.deb #Or go to https://grafana.com/grafana/download to get the latest version.
root@collector1:# dpkg  -i grafana-enterprise_9.3.1_amd64.deb
root@collector1:# systemctl daemon-reload
root@collector1:# systemctl start grafana-server
root@collector1:# systemctl status grafana-server
root@collector1:# systemctl enable grafana-server.service
root@collector1:# update-rc.d grafana-server defaults
root@collector1:# touch /var/log/telegraf/telegraf.log
root@collector1:# chmod go+rw /var/log/telegraf/telegraf.log
```
Login to the Grafana Server. It will ask to define your new password when login for the 1st time.
```bash
http://<ip-address-grafana-server>:3000
User: admin
Pass: admin
```
To verify if data is flowing from the controller to the collector on the defined port:
```bash
root@collector1:# tcpdump -i any port 57500
```
To verify if the data is flowing from the controller to the database on the defined port:
```bash
root@collector1:#  tcpdump -i any port 8086
```
