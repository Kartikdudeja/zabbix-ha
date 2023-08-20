# Zabbix 6.0 HA POC

## Getting Started

1. Run the Zabbix HA Test Lab Vagrantfile
``` bash
vagrant up
```

2. Install Mysql DB on DB Host:  
[Install MySQL on Rocky Linux 8](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-rocky-linux-8)

```bash
sudo dnf install mysql-server -y

sudo systemctl start mysqld.service && sudo systemctl enable mysqld.service

sudo mysql_secure_installation
```

> Note: Allow Remote Root Login for Mysql.

3. Install Zabbix on Zabbix Hosts:  
[Install Zabbix Proxy 6 on Rocky Linux 8](https://www.zabbix.com/download?zabbix=6.0&os_distribution=rocky_linux&os_version=8&components=server_frontend_agent&db=mysql&ws=apache)

- Install Zabbix Repository
``` bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/8/x86_64/zabbix-release-6.0-4.el8.noarch.rpm
sudo dnf clean all
```

- Install Zabbix server, frontend, agent
``` bash
sudo dnf install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent -y
```

- Create Intial Database

Note: DB Import is need to be done once

Make sure you have database server up and running. Run the following on your database host.

``` bash
mysql -uroot -ppassword
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@'%' identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@'%';
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```

On Zabbix server host import initial schema and data.
``` bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -h192.168.60.13 -uzabbix -ppassword zabbix
```

> Install Mysql Client using the following command: `sudo dnf install mysql -y`

Disable log_bin_trust_function_creators option after importing database schema
``` bash
mysql -uroot -ppassword
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

- Configure the database for Zabbix server

Edit file /etc/zabbix/zabbix_server.conf
``` bash
DBHost=192.168.60.13
DBPassword=password
```
- Start Zabbix server and agent processes
``` bash
sudo systemctl restart zabbix-server zabbix-agent httpd php-fpm
sudo systemctl enable zabbix-server zabbix-agent httpd php-fpm
```

- Open Zabbix UI web page
The default URL for Zabbix UI when using Apache web server is `http://host/zabbix`

4. Zabbix HA Cluster Configuration

Edit file /etc/zabbix/zabbix_server.conf

Host 01:
``` bash
HANodeName=zabbix-ha-server-01
NodeAddress=192.168.60.11:10051
```

Handy Commmand:
``` bash
grep -E "HANodeName=|NodeAddress=" /etc/zabbix/zabbix_server.conf

sed -i -e 's/^# HANodeName=/HANodeName=zabbix-ha-server-01/' -e 's/^# NodeAddress=localhost:10051/NodeAddress=192.168.60.11:10051/' /etc/zabbix/zabbix_server.conf
```

Host 02:
``` bash
HANodeName=zabbix-ha-server-02
NodeAddress=192.168.60.12:10051
```

Handy Commmand:
``` bash
grep -E "HANodeName=|NodeAddress=" /etc/zabbix/zabbix_server.conf

sed -i -e 's/^# HANodeName=/HANodeName=zabbix-ha-server-02/' -e 's/^# NodeAddress=localhost:10051/NodeAddress=192.168.60.12:10051/' /etc/zabbix/zabbix_server.conf
```

5. Zabbix HA Status Check
``` bash
zabbix_server -R ha_status
```

> Note: Make sure that Zabbix server address:port is not defined in the frontend configuration (found in conf/zabbix.conf.php of the frontend files directory).

``` bash
// Uncomment and set to desired values to override Zabbix hostname/IP and port.
// $ZBX_SERVER                  = '';
// $ZBX_SERVER_PORT             = '';
```

References:
https://www.zabbix.com/documentation/6.0/en/manual/concepts/server/ha

https://blog.zabbix.com/build-zabbix-server-ha-cluster-in-10-minutes-by-kaspars-mednis-zabbix-summit-online-2021/18155/#How_Zabbix_cluster_works

---

## Zabbix HA Keypoints:

- Zabbix 6 offers a native high-availability solution.
- Zabbix high availability mode multiple Zabbix servers are run as nodes in a cluster. While one Zabbix server in the cluster is active, others are on standby, ready to take over if necessary.
- uses the same database
- The ha manager process is responsible for checking the high availability node status in the database every 5 seconds and is responsible for taking over if the active node fails.
- For SNMP monitoring: make sure your endpoints accept connections from all of the Zabbix server cluster nodes
- Performance: The heartbeats that the cluster nodes send to the database backend are extremely small messages that get recorded in one of the smaller Zabbix database tables, so the performance impact should be negligible.

## Failover Mechanism:
- Only one node can be active (working) at a time. A standby node runs only one process - the HA manager. A standby node does no data collection, processing or other regular server activities; they do not listen on ports; they have minimum database connections.
- Both active and standby nodes update their last access time every 5 seconds. Each standby node monitors the last access time of the active node. If the last access time of the active node is over 'failover delay' seconds, the standby node switches itself to be the active node and assigns 'unavailable' status to the previously active node.
- The active node monitors its own database connectivity - if it is lost for more than failover delay-5 seconds, it must stop all processing and switch to standby mode. The active node also monitors the status of the standby nodes - if the last access time of a standby node is over 'failover delay' seconds, the standby node is assigned the 'unavailable' status.

## Zabbix HA Cluster Node Configuration Details:

Two parameters are required in the server configuration to start a Zabbix server as cluster node:

`HANodeName` parameter must be specified for each Zabbix server that will be an HA cluster node.  
This is a unique node identifier (e.g. zabbix-node-01) that the server will be referred to in agent and proxy configurations. If you do not specify `HANodeName`, then the server will be started in standalone mode.

`NodeAddress` parameter must be specified for each node.  
The `NodeAddress` parameter (address:port) will be used by Zabbix frontend to connect to the active server node. `NodeAddress` must match the IP or FQDN name of the respective Zabbix server.

``` bash
HANodeName=zabbix-ha-server-01
NodeAddress=192.168.60.11:10051
```

Restart all Zabbix servers after making changes to the configuration files. They will now be started as cluster nodes.

### Status of the servers can be seen by running the command:
``` bash
zabbix_server -R ha_status
```

## Preparing frontend
Make sure that Zabbix server address:port is not defined in the frontend configuration (found in `conf/zabbix.conf.php` of the frontend files directory).

``` bash
// Uncomment and set to desired values to override Zabbix hostname/IP and port.
// $ZBX_SERVER                  = '';
// $ZBX_SERVER_PORT             = '';
```

## Agent configuration
HA cluster nodes (servers) must be listed in the configuration of Zabbix agent or Zabbix agent 2.

To enable passive checks, the node names must be listed in the `Server` parameter, separated by a comma.
``` bash
Server=zabbix-node-01,zabbix-node-02
```

To enable active checks, the node names must be listed in the `ServerActive` parameter. Note that for active checks the nodes must be separated by a comma from any other servers, while the nodes themselves must be separated by a semicolon, e.g.:
``` bash
ServerActive=zabbix-node-01;zabbix-node-02
```

> comma-separated list for passive Zabbix agents and nodes separated by semicolons for active Zabbix agents!

## Failover to standby node
Zabbix will fail over to another node automatically if the active node stops. There must be at least one node in standby status for the failover to happen.

How fast will the failover be? All nodes update their last access time (and status, if it is changed) every 5 seconds. So:

If the active node shuts down and manages to report its status as "stopped", another node will take over within 5 seconds.

If the active node shuts down/becomes unavailable without being able to update its status, standby nodes will wait for the failover delay + 5 seconds to take over

The failover delay is configurable, with the supported range between 10 seconds and 15 minutes (one minute by default). To change the failover delay, you may run:
``` bash
zabbix_server -R ha_set_failover_delay=5m
```
