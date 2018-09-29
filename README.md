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
        1. systemctl start rabbitmq-server.service
    3. `rabbitmqctl add_user openstack RABBIT_PASS`  (Replace RABBIT_PASS with a suitable password.)
    4. `rabbitmqctl set_permissions openstack ".*" ".*" ".*"` (Permit configuration, write, and read access for the openstack user:)

6. memcached
    1. yum install memcached python-memcached
    2. Edit the `/etc/sysconfig/memcached` file and complete the following actions:
    Configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:
    OPTIONS="-l 127.0.0.1,::1,controller"
    3. systemctl enable memcached.service
        systemctl start memcached.service

7. Etcd
    1. yum install etcd
    2. Edit the `/etc/etcd/etcd.conf` file and set the `ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS` to the management IP address of the controller node to enable access by other nodes via the management network:
    ```python
    #[Member]
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    ETCD_LISTEN_PEER_URLS="http://10.0.0.11:2380"
    ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
    ETCD_NAME="controller"
    #[Clustering]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
    ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ```
    3. script
    ```python
    systemctl enable etcd
    systemctl start etcd
    ```

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
5. `su -s /bin/sh -c "keystone-manage db_sync" keystone`
6. Initialize Fernet key repositories:
```python
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
7. Bootstrap the Identity service:
```python
keystone-manage bootstrap --bootstrap-password lizhixuan123 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

# Replace ADMIN_PASS with a suitable password for an administrative user.
```
8. Configure the Apache HTTP server¶
    https://docs.openstack.org/keystone/queens/install/keystone-install-rdo.html

    1. Edit the `/etc/httpd/conf/httpd.conf` file and configure the ServerName option to reference the controller node:
    ```python
    ServerName controller
    ```
    2. `ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/`
    3. installation
    ```python
    systemctl enable httpd.service
    systemctl start httpd.service
    ```
    4. Configure the administrative account
    ```python
    export OS_USERNAME=admin
    export OS_PASSWORD=lizhixuan123
    export OS_PROJECT_NAME=admin
    export OS_USER_DOMAIN_NAME=Default
    export OS_PROJECT_DOMAIN_NAME=Default
    export OS_AUTH_URL=http://controller:35357/v3
    export OS_IDENTITY_API_VERSION=3
    # Replace ADMIN_PASS with the password used in the keystone-manage bootstrap command in keystone-install-configure-rdo.
    ```

## Create a domain, projects, users, and roles
https://docs.openstack.org/keystone/queens/install/keystone-users-rdo.html

1. openstack domain create --description "An Example Domain" example
2. openstack project create --domain default --description "Service Project" service
3. openstack project create --domain default --description "Demo Project" demo
4. openstack user create --domain default   --password-prompt demo
5. openstack role create user
6. openstack role add --project demo --user demo user

## Verify operation
https://docs.openstack.org/keystone/queens/install/keystone-verify-rdo.html
1. unset OS_AUTH_URL OS_PASSWORD
2. As the admin user, request an authentication token:
```python
openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```
3. As the demo user, request an authentication token:
```python
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue
```


## Creating the scripts¶
https://docs.openstack.org/keystone/queens/install/keystone-openrc-rdo.html

1. Create and edit the `admin-openrc` file and add the following content:
```python
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=lizhixuan123
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
2. Create and edit the `demo-openrc` file and add the following content:
```python
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=lizhixuan123
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
3. Using the scripts¶
    1. `. admin-openrc`
    2. Request an authentication token:
    ```python
    openstack token issue
    ```





# Image service – glance installation for Queens

1. mysql -u root -p
2. mysql_script
```python
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'  IDENTIFIED BY 'GLANCE_DBPASS';
```
3. Source the admin credentials to gain access to admin-only CLI commands:
```python
. admin-openrc
```

4. openstack user create --domain default --password-prompt glance
5. openstack role add --project service --user glance admin
6. openstack service create --name glance   --description "OpenStack Image" image
7. openstack endpoint create --region RegionOne   image public http://controller:9292
8. openstack endpoint create --region RegionOne   image internal http://controller:9292
9. openstack endpoint create --region RegionOne   image admin http://controller:9292

10. yum install openstack-glance
11. Edit the `/etc/glance/glance-api.conf` file and complete the following actions:
In the [database] section, configure database access:
```python
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

```
12. Edit the `/etc/glance/glance-registry.conf` file and complete the following actions:
```python
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone

```

13. su -s /bin/sh -c "glance-manage db_sync" glance
14. run_script
```python
systemctl enable openstack-glance-api.service  openstack-glance-registry.service
systemctl start openstack-glance-api.service   openstack-glance-registry.service
```

