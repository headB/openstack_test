# environment

1. security
Passwords¶

|Password name	    |   Description|
|-------------------|--------------|
|Database password           	|Root password for the database|
|ADMIN_PASS	                    |Password of user admin|
|CINDER_DBPASS	                |Database password for the Block Storage service
|CINDER_PASS	                  |  Password of Block Storage service user cinder|
|DASH_DBPASS	                 |   Database password for the Dashboard|
|DEMO_PASS	                    |Password of user demo|
|GLANCE_DBPASS	                |Database password for Image service|
|GLANCE_PASS	                   | Password of Image service user glance|
|KEYSTONE_DBPASS	              |  Database password of Identity service|
|METADATA_SECRET	             |   Secret for the metadata proxy|
|NEUTRON_DBPASS	                |Database password for the Networking service|
|NEUTRON_PASS	                |Password of Networking service user neutron|
|NOVA_DBPASS	                 |   Database password for Compute service|
|NOVA_PASS	                    |Password of Compute service user nova|
|PLACEMENT_PASS	                |Password of the Placement service user placement|
|RABBIT_PASS	                   | Password of RabbitMQ user openstack|

2. HOST network None

3. install openstack package
    1. yum install centos-release-openstack-queens
    2. yum upgrade
    3. yum install python-openstackclient
    4. yum install openstack-selinux (option)

4. SQL
    1. yum install mariadb mariadb-server python2-PyMySQL
    2. edit /etc/my.conf.d/openstack.conf
    ```python
    [mysqld]
    bind-address = 10.0.0.11
    default-storage-engine = innodb
    innodb_file_per_table = on
    max_connections = 4096
    collation-server = utf8_general_ci
    character-set-server = utf8
    ```
    3. systemctl enable mariadb.service
    4. systemctl start mariadb.service
    5. mysql_secure_installation

5. Message Queue
    1. yum install rabbitmq-server
    2.  systemctl enable rabbitmq-server.service
        systemctl start rabbitmq-server.service
    3. rabbitmqctl add_user openstack RABBIT_PASS  (Replace RABBIT_PASS with a suitable password.)
    4. rabbitmqctl set_permissions openstack ".*" ".*" ".*" (Permit configuration, write, and read access for the openstack user:)

6. Etcd
    1. yum install memcached python-memcached
    2. Edit the /etc/sysconfig/memcached file and complete the following actions:
    Configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:
    OPTIONS="-l 127.0.0.1,::1,controller"
    3. systemctl enable memcached.service
        systemctl start memcached.service

# install minideployment for Queens(Controller node)

## identity service
1. mysql -u root -p
2. create database
```python
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'lizhixuan123!';

#Replace KEYSTONE_DBPASS with a suitable password.
```
3. yum install openstack-keystone httpd mod_wsgi
4. edit
```python
Edit the /etc/keystone/keystone.conf file and complete the following actions:
In the [database] section, configure database access:
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
[token]
# ...
provider = fernet
```
5. su -s /bin/sh -c "keystone-manage db_sync" keystone
6. 
