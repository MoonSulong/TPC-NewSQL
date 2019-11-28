# TPC-NewSQL
Benchmark NewSQL(MemSQL) database performance based on TPCC protocol.

* [MemSQL](https://docs.memsql.com/v6.8/introduction/documentation-overview/) offical documents
* [TPC-C Benchmark](https://github.com/Percona-Lab/tpcc-mysql) Percona-Lab tpcc-mysql tutorial

## Environment Setup

This experiment is set on 3 nodes cluster(Ubuntu 16.04), and MySQL is used to transfer data generated from TPC-C.

### Update and upgrade Ubuntu 16.04

```
sudo apt update
sudo apt upgrade
```
### MySQL 

#### Install MySQL Server and Client (Skip passord set for root user)
```
sudo apt install mysql-server
sudo apt install mysql-client

# Install `mysql-config`
sudo apt-get install libmysqlclient-dev

# Open Mysql Console
mysql -p -u root

# In case service is not started
sudo service mysql start 
```

### TPC-C data 

```
git clone https://github.com/Percona-Lab/tpcc-mysql
```

#### Build Binaries

Running `make -C src/` will build two binaries: `tpcc_load` and `tpcc_start`.

#### Generate Dataset

* Create database
     `mysqladmin create tpcc1000`
   * create tables
     `mysql tpcc1000 < create_table.sql`
   * create indexes and FK ( this step can be done after loading data)
     `mysql tpcc1000 < add_fkey_idx.sql`
   * populate data
     - simple step
       `tpcc_load -h127.0.0.1 -d tpcc1000 -u root -p "" -w 1000`
                 |hostname:port| |dbname| |user| |password| |WAREHOUSES|
       ref. tpcc_load --help for all options
     - load data in parallel 
       check load.sh script
* Start benchmark
   * `./tpcc_start -h127.0.0.1 -P3306 -dtpcc1000 -uroot -w1000 -c32 -r10 -l10800` (port can be modified into 3307 for MemSQL)
   * |hostname| |port| |dbname| |user| |WAREHOUSES| |CONNECTIONS| |WARMUP TIME| |BENCHMARK TIME|
   * ref. tpcc_start --help for all options 

### MemSQL 

#### SSH setup
private/public key pair using 'ssh-keygen -t rsa' on the master node 
Copy the public key of node-0 to the authorized_keys file in all the nodes(including node-0) under ~/.ssh/. 
To get the content of the public key, do 'cat ~/.ssh/id_rsa.pub'
	
#### Install MemSQL
```
# 1. Install online
wget -O - 'https://release.memsql.com/release-aug2018.gpg'  2>/dev/null | sudo apt-key add - && apt-key list

# 2.1 Verify
apt-cache policy apt-transport-https

# 2.2 If 'apt-transport-https' is not installed, you must install it before proceeding(optional)
sudo apt -y install apt-transport-https

# 3 Add the MemSQL repository to retrieve its packages.
echo "deb [arch=amd64] https://release.memsql.com/production/debian memsql main" | sudo tee /etc/apt/sources.list.d/memsql.list

# 4 Install finish
sudo apt update && sudo apt -y install memsql-toolbox memsql-client memsql-studio
```

#### Deploy MemSQL
````
# Deploy and change port
memsql-deploy setup-cluster -i /path/to/yourSSHkey --license [YOUR LICENSE KEY] \
    --master-host <main_IP_address> \
    --aggregator-hosts <child_agg_IP_address> \
    --leaf-hosts <leaf1_IP_address>,<leaf2_IP_address> \
	--memsql-port 3307 #(important! modify the port for MemSQL because MySQL use the same port)
    --password <secure_password> # reset latter to no password 

# Optimize
memsql-admin optimize
````
#### Start MemSQL Studio
````
sudo systemctl start memsql-studio
sudo memsql-studio &

# Check process
sudo lsof -i -P -n | grep LISTEN
````

* Goto http://<main_deployment_machine>:8080 and click Add New Cluster to setup a cluster.
* Paste the main deployment machine IP address into Hostname.
* Set Port to 3307.
* Specify root as the Username.
* Enter the Password you set in the setup-cluster step from the previous page.
* Click Create Cluster Profile and set Type as Development.
* Fill in Cluster Name and Description to your preference.

#### Change root-password into empty
````
sudo memsqlctl change-root-password --password ""
+-------+------------+------------+------+---------------+---------+
| Index | MemSQL ID  |    Role    | Port | Process State | Version |
+-------+------------+------------+------+---------------+---------+
| 1     | 01FA0ABD58 | Aggregator | 3306 | Running       | 6.5.10  |
| 2     | 994274A024 | Leaf       | 3307 | Running       | 6.5.10  |
| 3     | All Nodes  |            |      |               |         |
+-------+------------+------------+------+---------------+---------+
Select an option: 2
memsqlctl will perform the following actions
  · On MemSQL node with ID 994274A024996ADAD6B1B780352C0EDBC0E7328F:
    - Run `SET PASSWORD FOR 'root'@'%' = PASSWORD()`

Would you like to continue? [y/N]: y
✓ Set new password for node with MemSQL ID 994274A024996ADAD6B1B780352C0EDBC0E7328F
````

### Transitioning from MySQL to MemSQL
````
# Output schema and data
$ mysqldump -h 127.0.0.1 -u root -B [database name] --no-data -r schema.sql
$ mysqldump -h 127.0.0.1 -u root -B [database name] --no-create-info -r data.sql

# Replay
$ mysql -h 127.0.0.1 -u root -P 3307 < schema.sql
$ mysql -h 127.0.0.1 -u root -P 3307 < data.sql
````


Troubleshooting Guide:

Q: MySQL database access permission denied

A: https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost
````

$ sudo mysql -u root

mysql> USE mysql;
mysql> SELECT User, Host, plugin FROM mysql.user;
mysql> USE mysql;
mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
mysql> FLUSH PRIVILEGES;
mysql> exit;

$ service mysql restart
````

Q: MySQL cannot connect to MemSQL

A: Make sure no passwords for both of them.

Q: MemSQL studio port cannot access if use remote node  

A: Can redirect reomote port to local port 


