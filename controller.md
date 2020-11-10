# prerquisites
1. mysql -u root -p
2. run
```python
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;


GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'  IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%'   IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'   IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'  IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost'   IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%'  IDENTIFIED BY 'lizhixuan123!';
```

3. `. admin-openrc`
4. openstack user create --domain default --password-prompt nova
5. openstack role add --project service --user nova admin
6. openstack service create --name nova  --description "OpenStack Compute" compute
7. `openstack endpoint create --region RegionOne  compute public http://controller:8774/v2.1`
8. `openstack endpoint create --region RegionOne  compute internal http://controller:8774/v2.1`
9. `openstack endpoint create --region RegionOne   compute admin http://controller:8774/v2.1`
10. openstack user create --domain default --password-prompt placement
11. openstack role add --project service --user placement admin
12. openstack service create --name placement --description "Placement API" placement
13. `openstack endpoint create --region RegionOne placement public http://controller:8778`
14. `openstack endpoint create --region RegionOne placement internal http://controller:8778`
15. `openstack endpoint create --region RegionOne placement admin http://controller:8778`
16. yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api
17. Edit the /etc/nova/nova.conf file and complete the following actions:
In the [DEFAULT] section, enable only the compute and metadata APIs:
```python
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 10.0.0.11
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = lizhixuan123
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://controller:9292
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = lizhixuan123

```
18. Due to a packaging bug, you must enable access to the Placement API by adding the following configuration to /etc/httpd/conf.d/00-nova-placement-api.conf:
```python
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```
19. systemctl restart httpd
20. su -s /bin/sh -c "nova-manage api_db sync" nova
21. su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
22. su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
23. su -s /bin/sh -c "nova-manage db sync" nova
24. nova-manage cell_v2 list_cells
25. run_script
```python

systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

#重启命令
systemctl restart openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

```
-------------------------------
# network 
https://docs.openstack.org/neutron/queens/install/controller-install-rdo.html#prerequisites
1. mysql -u root -p
2. mysql_script
```python
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'   IDENTIFIED BY 'lizhixuan123!';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%'   IDENTIFIED BY 'lizhixuan123!';
```
3. `. admin-openrc`
4. openstack user create --domain default --password-prompt neutron
5. openstack role add --project service --user neutron admin
6. openstack service create --name neutron   --description "OpenStack Networking" network
7. openstack endpoint create --region RegionOne  network public http://controller:9696
8. openstack endpoint create --region RegionOne  network internal http://controller:9696
9. openstack endpoint create --region RegionOne   network admin http://controller:9696

----------------------------------------
##select network type

Choose one of the following networking options to configure services specific to it. Afterwards, return here and proceed to Configure the metadata agent.

Networking Option 1: Provider networks
Networking Option 2: Self-service networks
--------------------------------
### use option 1
https://docs.openstack.org/neutron/queens/install/controller-install-option1-rdo.html
1. yum install openstack-neutron openstack-neutron-ml2  openstack-neutron-linuxbridge ebtables
2. Edit the /etc/neutron/neutron.conf file and complete the following actions:
In the [database] section, configure database access:
```python
[database]
# ...
connection = mysql+pymysql://neutron:lizhixuan123!@controller/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = lizhixuan123

[nova]
# ...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = lizhixuan123

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp

```

3. Edit the `/etc/neutron/plugins/ml2/ml2_conf.ini` file and complete the following actions:
In the [ml2] section, enable flat and VLAN networks:
```python
[ml2]
# ...
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[securitygroup]
# ...
enable_ipset = true

```

4. Edit the `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` file and complete the following actions:
In the [linux_bridge] section, map the provider virtual network to the provider physical network interface:
```python

[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = false

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver



```

5. Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1:
```python
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

6. Edit the /etc/neutron/dhcp_agent.ini file and complete the following actions:
In the [DEFAULT] section, configure the Linux bridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network:
```python
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true

```

-------------------------------------
# continue
1. Edit the `/etc/neutron/metadata_agent.ini` file and complete the following actions:
In the [DEFAULT] section, configure the metadata host and shared secret:
```python

[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = lizhixuan123
#Replace METADATA_SECRET with a suitable secret for the metadata proxy.

```
2. Edit the `/etc/nova/nova.conf` file and perform the following actions:
In the [neutron] section, configure access parameters, enable the metadata proxy, and configure the secret:
```python
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = lizhixuan123
#Replace NEUTRON_PASS with the password you chose for the neutron user in the Identity service.

#Replace METADATA_SECRET with the secret you chose for the metadata proxy.

```

3. ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
4. su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf   --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
5. systemctl restart openstack-nova-api.service
6. run_script
```python
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service

systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
  
#重启命令
systemctl restart neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service

  
  
```
7. systemctl enable neutron-l3-agent.service
8. systemctl start neutron-l3-agent.service
