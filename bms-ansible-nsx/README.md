# preparing your Bare Metal Server
yum install -y tcpdump boost-filesystem PyYAML boost-iostreams boost-chrono python-mako python-netaddr python-six gperftools-libs libunwind libelf-dev snappy boost-date-time c-ares redhat-lsb-core wget net-tools yum-utils lsof python-gevent libev python-greenlet libvirt-libs ansible git


# Ansbile for Bare Metal Server
Ansible for Bare Metal Server is a set of automated scripts to setup Application 
Interface for Bare Metal Server;

configure hosts file 
vi hosts

please enter all the info needed : 
for example : 

# Create an group that contains the Bare Metal Servers
[TransportNodes:children]
servers_static
servers_dhcp
servers_migration

# Set variables common for all Bare Metal Servers
[TransportNodes:vars]
# SSH user
ansible_ssh_user=root
ansible_ssh_pass=VMware1!

rhel_dependency=["tcpdump", "boost-filesystem", "PyYAML", "boost-iostreams", "boost-chrono", "python-mako", "python-netaddr", "python-six", "gperftools-libs", "libunwind", "snappy", "boost-date-time", "c-ares", "redhat-lsb-core", "wget", "net-tools", "yum-utils", "lsof", "libvirt-libs", "python-gevent", "libev", "python-greenlet"]

ubuntu16_dependency=["libunwind8", "libgflags2v5", "libgoogle-perftools4", "traceroute", "python-mako", "python-simplejson", "python-unittest2", "python-yaml", "python-netaddr", "libboost-filesystem1.58.0", "libboost-chrono1.58.0", "libgoogle-glog0v5", "dkms", "libboost-date-time1.58.0", "python-protobuf", "python-gevent", "libsnappy1v5", "libleveldb1v5", "libboost-program-options1.58.0", "libboost-thread1.58.0", "libboost-iostreams1.58.0", "libvirt0", "libelf-dev"]

ubuntu18_dependency=["traceroute", "python-mako", "python-netaddr", "python-simplejson", "python-unittest2", "python-yaml", "python-openssl", "dkms", "libvirt0", "libelf-dev"]

suse12_dependency=["net-tools", "tcpdump", "python-simplejson", "python-netaddr", "python-PyYAML", "python-six", "libunwind", "wget", "libvirt-libs", "lsof", "libcap-progs"]

# host group for servers
[servers_static]
localhost ansible_ssh_host=localhost static_ip=10.11.11.20 netmask=255.255.255.0 ls_name=web-centos

[servers_dhcp]

[servers_migration]
#server6 ansible_ssh_host= migrate_intf= ls_name=

[servers_restoration]
#server6

# NSX Configuration
[NSX]
#============================
# NSX Manager Credential
nsxmanager ip=192.168.110.10 username=admin password=1234! thumbprint=14437c1c871229c6fe4c7afc3ebf755b74

# Ansible - Support Methods Modes
1. Static  
Enable static configuration on Application Interface;

2. Dhcp  
Enable dhcp configuration on Application Interface;

3. Migration  
This mode supports Management and Application sharing the same IP;  
Enable migration mode on Application Interface; Also named as “underlay mode” or
"VLAN-0 mode";

# WARNING
For this configure script, it will call NSX Manager REST APIs; To authenticate a request using HTTP Basic authentication,
user need to provider username and password in Ansible Inventory file(hosts) for authentication;

From security perspective, PLS. NOTE this action and protect username and password by themselves;

# Usage

### 1. Static
#### Host group name: servers_static

#### Configuration Parameters

Parameter | requirement | description | example
---|---|---|---
static_ip | required | static ip address of app | static_ip=192.1.1.21
netmask | required | netmask | netmask=255.255.255.0
ls_id or ls_name | required | logical switch id or name | ls_id=dc97de40-b476-4698-a590-0a8c60100ac5 or ls_name=vlan_ls
mac_address | optional | mac address of app's interface, random by default | mac_address=11:22:33:44:55:66
app_intf_name | optional | app interface name, "nsx-eth" by default | app_intf_name="nsx-eth"
routing_table | optional | routing table;  | routing_table='["-net 101.20.0.0/16", "-net 100.20.20.0/24"]'

#### Command
```bash
$ansible-playbook -i hosts static_config.yml
```

#### routing table syntax
Iface of each entry in routing table is app_intf_name.
```bash
[-net|-host] target [netmask Nm] [gw Gw] [metric N] i [mss M] [window W] [irtt m] [reject] [mod] [dyn] [reinstate]
```

### 2. DHCP
#### Host group name: servers_dhcp

#### Configuration Parameters

|Parameter | requirement | description| example |
|---|---|---|---|
|ls_id or ls_name | required | logical switch id or name | ls_id=dc97de40-b476-4698-a590-0a8c60100ac5 or ls_name=vlan_ls |
|mac_address | optional | mac address of app's interface, random by default| mac_address="11:22:33:44:55:66 |
|app_intf_name | optional | app interface name, "nsx-eth" by default| app_intf_name="nsx-eth" |
|routing_table | optional | routing table | routing_table='["-net 101.20.0.0/16", "-net 100.20.20.0/24"]' |

#### Command
```bash
$ansible-playbook -i hosts dhcp_config.yml
```

### 3. Migration
#### Host group name: servers_migration

#### Configuration Parameters

Parameter | requirement | description | example
---|---|---|---
ls_id or ls_name | required | logical switch id or name | ls_id=dc97de40-b476-4698-a590-0a8c60100ac5 or ls_name=vlan_ls
migrate_intf | required | interface name needs to be migrated | migrate_intf="eth0"
app_intf_name | optional | app interface name, "nsx-eth" by default | app_intf_name="nsx-eth"

#### Command
```bash
$ansible-playbook -i hosts migrate.yml
```

### 4. Restoration
```bash
$ansible-playbook -i hosts restore.yml
```

### 5. Bootstrap flow in underlay mode
In underlay mode, since management and application traffic share a single IP, we need a way to differentiate the forwarding of the types of traffic, and the management traffic should bypass NSX logical pipeline, otherwise any misconfiguration of NSX networking and policies will cause connectivity loss of the Bare Metal server and could not be recovered.  
So here we add high priority flows(classified by remote IP, proto, local/remote ports) for the management traffic when setup application interface, we call high priority flow as bootstrap flow;
NSX Manager and Controller endpoints will be loaded in bootstrap flow by default;
If Manager/Controller endpoint is FQDN, that won't support bootstrap flow.

If User want to add/delete/update bootstrap flow, pls. follow below steps:
```bash
1. update "templates/nsx-baremetal.j2";
2. $ansible-playbook -i hosts config/bms_update.yml 
```


### 6. Prepare host with dependency package
```bash
$ansible-playbook -i hosts prepare.yml
```
### 7. Remove dependency which are installed during prepare phase
```bash
$ansible-playbook -i hosts unprepare.yml
```
