#Install the packages
https://docs.openstack.org/nova/queens/install/compute-install-rdo.html#install-and-configure-components
1. yum install openstack-nova-compute
2. Edit the /etc/nova/nova.conf file and complete the following actions:
In the [DEFAULT] section, enable only the compute and metadata APIs:
### 下面是是第一份配置,写在 /etc/nova/nova.conf
```python

[DEFAULT]

enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:lizhixuan123@controller

my_ip = #需要你,填写本机ip
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]

auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = lizhixuan123

[vnc]

enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]

api_servers = http://controller:9292

[oslo_concurrency]

lock_path = /var/lib/nova/tmp

[placement]

os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = lizhixuan123


```

3. egrep -c '(vmx|svm)' /proc/cpuinfo
4. 注意,这一步,除非不知道硬件的虚拟化,才设置这个地方! Edit the [libvirt] section in the /etc/nova/nova.conf file as follows:
```python
[libvirt]

virt_type = qemu
```
5. systemctl enable libvirtd.service openstack-nova-compute.service
6. systemctl start libvirtd.service openstack-nova-compute.service

---------

## Add the compute node to the cell database¶
#### Important
#### Run the following commands on the controller node.
1. . admin-openrc
2. su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova


# network
https://docs.openstack.org/neutron/queens/install/compute-install-rdo.html#install-the-components

1. yum install openstack-neutron-linuxbridge ebtables ipset
2. Edit the /etc/neutron/neutron.conf file and complete the following actions:
### 下面是是第二份配置,写在 /etc/neutron/neutron.conf (不要忘记配置我,不然后果很严重)
```python
[DEFAULT]

transport_url = rabbit://openstack:lizhixuan123@controller
auth_strategy = keystone

[keystone_authtoken]

auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = lizhixuan123

[oslo_concurrency]

lock_path = /var/lib/neutron/tmp

```
## config network 

## select option 1
1. Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file and complete the following actions:
```python
[linux_bridge]
physical_interface_mappings = provider:#需要填写,属于管理网段的ip地址

[vxlan]
enable_vxlan = false

[securitygroup]

enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

```
2. Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1:
```python
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

## continue
1. Edit the /etc/nova/nova.conf file and complete the following actions:
```python
[neutron]

url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = lizhixuan123
```
2. systemctl restart openstack-nova-compute.service
3. systemctl enable neutron-linuxbridge-agent.service
4. systemctl start neutron-linuxbridge-agent.service

```
 Note

If the nova-compute service fails to start, check /var/log/nova/nova-compute.log. The error message AMQP server on controller:5672 is unreachable likely indicates that the firewall on the controller node is preventing access to port 5672. Configure the firewall to open port 5672 on the controller node and restart nova-compute service on the compute node.
```

------------------
---------------------

# Add the compute node to the cell database¶
1. . admin-openrc
2. su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
3. 
