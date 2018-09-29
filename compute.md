#Install the packages
https://docs.openstack.org/nova/queens/install/compute-install-rdo.html#install-and-configure-components
1. yum install openstack-nova-compute
2. Edit the /etc/nova/nova.conf file and complete the following actions:
In the [DEFAULT] section, enable only the compute and metadata APIs:
```python

[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:lizhixuan123@controller
#RABBIT_PASS
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
#Replace MANAGEMENT_INTERFACE_IP_ADDRESS with the IP address of the management network interface on your compute node, typically 10.0.0.31 for the first node in the example architecture.
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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
# ...
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
# ...
api_servers = http://controller:9292

oslo_concurrency]
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

3. egrep -c '(vmx|svm)' /proc/cpuinfo
4. Edit the [libvirt] section in the /etc/nova/nova.conf file as follows:
```python
[libvirt]
# ...
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
```python
[DEFAULT]
# ...
transport_url = rabbit://openstack:lizhixuan123@controller
auth_strategy = keystone

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

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp

```
## config network 

## select option 1
1. Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file and complete the following actions:
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
2. Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1:
```python
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
```

## continue
1. Edit the /etc/nova/nova.conf file and complete the following actions:
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
password = lizhixuan123
```
2. systemctl restart openstack-nova-compute.service
3. systemctl enable neutron-linuxbridge-agent.service
4. systemctl start neutron-linuxbridge-agent.service
