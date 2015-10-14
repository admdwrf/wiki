# Install and configure OpenStack Kilo on CentOS 7 with RDO project.
![](/labs/openstack/img/logo.jpg)
## Goal of this lab
I needed install an OpenStack infra to learn each bricks of this solution.
For this first install, I installed OpenStack services in just one node.
It was perfect to me to discover OpenStack technologies.


Note: This install is to test OpenStack, not for a production environment.

## OS and softwares we will use
**Hardware:** I used a VMware VM under ESXi 6. I configured it with 8 cores, 12 Go of RAM and 500 gb of disk space.

**Os:** GNU/linux CentOS 7

## Network details

| Network informations       | Value          |
| ------------- |-------------|
| Network adress      | 172.30.242.0/24 |
| Network Gateway      | 172.30.242.254      |
| DNS Servers | 8.8.8.8,8.8.4.4      |
| Domain | fredalix.lab      |
| VM Hostname | rdo      |
| VM IP | 172.30.242.42      |
| MAC | 00:0c:29:ef:b5:da      |

# Pre requisite

Into my lab network, I don't have DHCP server.

## Create VMware VM

### Configuration

![](/labs/openstack/img/rdo-00.png)

### Add option to run KVM under a VMware VM

So how can vmware ESXi 5.5 or 6 be instructed to provide the CPU virtualization vmx flag to the guest hypervisor without web client ?
Steps:
1. Shutdown the VM
2. Locate the guest hypervisor virtual machine configuration file (*VM-name.vmx*), edit and add the following line at the end:
```
vhv.enable = "TRUE"
```
3. Boot the VM

# Install and configure CentOS 7
## Installation
### Disk partition

![](/labs/openstack/img/rdo-02.png)
![](/labs/openstack/img/rdo-03.png)
By default, the parition /home have the biggest space allocated. We'll need to have this space for the / partition.
In Installation Destination, at the "Other Storage Options/Partitioning" select "I will configure partitioning" and click on "Done" button.
Reduce the /home space allocation and increase the / partition.


### Software Selection
![](/labs/openstack/img/rdo-08.png)
![](/labs/openstack/img/rdo-01.png)
In "Basic Environment" area, use the default "Minimal install" and in "Adds-Ons for Selected Environment", select "Developments Tools".

### Network configuration
![](/labs/openstack/img/rdo-04.png)
![](/labs/openstack/img/rdo-05.png)
![](/labs/openstack/img/rdo-06.png)

In "Network & Hostname", set the server's hostname (in this case rdo.fredalix.lab) and click on "Configure" button.
Set the ip adress in ipv4 section, set the dns server and the domain name.
After enable the network interface

### Configure time with ntp
![](/labs/openstack/img/rdo-07.png)
In "Date & Time" enable "network time"

### Other
![](/labs/openstack/img/rdo-11.png)
![](/labs/openstack/img/rdo-10.png)
![](/labs/openstack/img/rdo-09.png)

Set the root password, create the user admin and set its password too.

## Configure

## Disable NetworkManager
```
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl enable network
```

## Modify /etc/hosts
warning: This step is very important, or you'll can't deploy OpenStack

Set correct /etc/hosts with your ip, hostname, hostname.domain
In my case it's **172.30.242.42   rdo rdo.fredalix.lab**
```
sudo vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.30.242.42   rdo rdo.fredalix.lab
```

## Disable SELinux
If you have SELinux enabled, disable it - it interferes with everything.
```
sudo vi /etc/sysconfig/selinux

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 
```

We'll reboot the server later to modification take effect.

## Install additionals packages
We need to install python-dev packages.
By default, the CentOS 7 minimal install don't install ifconfig command. Install net-tools package to have this command.
```
sudo yum -y install git python-devel python-pip net-tools
```

## Update OS
Update CentOS 7 with latest packages and reboot the server
```
sudo yum -y update
sudo reboot
```

# Install Openstack

## Install rdo packages
Set the rdo repos
```
sudo yum install -y https://rdoproject.org/repos/rdo-release.rpm
```

Now, install the packstack installer
```
sudo yum install -y openstack-packstack
```

## Launch Openstack installation
Now, it's time to install Openstack ! :)
```
packstack --allinone --provision-demo=n
```
This operation will take few minutes. You have time to drink a good coffee :)

# Configure Openstack


## Configure OVS external bridge
Create OVS bridge interface by creating file /etc/sysconfig/network-scripts/ifcfg-br-ex
```
sudo sh -c "/usr/bin/echo '
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
MACADDR=00:0c:29:ef:b5:da
BOOTPROTO=static
IPADDR=172.30.242.42
NETMASK=255.255.255.0
GATEWAY=172.30.242.254
DNS1=8.8.8.8
ONBOOT=yes' > /etc/sysconfig/network-scripts/ifcfg-br-ex"
```

Configure your physical network interface for OVS bridging in /etc/sysconfig/network-scripts/ifcfg-<name>
```
sudo sh -c "/usr/bin/echo '
DEVICE=eno16777984
HWADDR=00:0c:29:ef:b5:da
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes' > /etc/sysconfig/network-scripts/ifcfg-eno16777984"
```

Modify Opentack parameters
```
sudo openstack-config --set /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini ovs bridge_mappings extnet:br-ex
sudo openstack-config --set /etc/neutron/plugin.ini ml2 type_drivers vxlan,flat,vlan
```

Restart network services
```
sudo service network restart
sudo service neutron-openvswitch-agent restart
sudo service neutron-server restart
```

## Configure Openstack network
Load the openstack admin user profile
```
source keystonerc_admin
```

Create the external network
```
neutron net-create external_network --provider:network_type flat --provider:physical_network extnet  --router:external --shared
```

Create the public network
```
neutron subnet-create --name public_subnet --enable_dhcp=False --allocation-pool=start=172.30.242.150,end=172.30.242.200 \
                        --gateway=172.30.242.254 external_network 172.30.242.0/24 --dns-nameservers list=true 8.8.8.8 8.8.4.4
neutron router-create router1
neutron router-gateway-set router1 external_network
```

Create the private network
```
neutron net-create private_network
neutron subnet-create --name private_subnet private_network 192.168.242.0/24
```

Connect the private network to the public network via the router router1
```
neutron router-interface-add router1 private_subnet
```

## Manage security groups
List security group list
```
neutron security-group-list
```

Normally you'll see juste one security group, in the case you have two security group list with the name "default"
Delete the first.
```
neutron security-group-delete 80ee9d7f-0e2f-4771-9b4d-22c6b65f41e9
```

Now, auhorize ICMP and sshp protocol to ping and connect over ssh to your futures instances.
```
neutron security-group-rule-create --protocol icmp --direction ingress default
neutron security-group-rule-create --protocol tcp --port-range-min 22 --port-range-max 22 --direction ingress default
```

## Import your ssh key
To permit to connect in your instances, you need to import your ssh key.
In this case, I put my own ssh key into /var/tmp directory
```
nova keypair-add --pub-key /var/tmp/id_rsa.pub my-key
```

## Create a floating IP address
We need to allocate IP address on our public network to connect to our futures instances.
```
nova floating-ip-create external_network
```

## Import image
For our first instance, we'll deploy cirros
```
curl http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img | glance \
         image-create --name='cirros image' --is-public=true  --container-format=bare --disk-format=qcow2
```

# Create and connect in our first instance

## Create the instance
First, get Image'ID instance you want use.
```
nova image-list
```

You need the private_network ID
```
nova network-list
```

Create and launch the instance.
```
nova boot --flavor 1 --nic net-id=ec8146b3-1525-4408-b895-900b695747f9 --key_name my-key --image 39018bb0-d883-404e-99c8-6d4f8b175f49 --security_group default cirrOS
```

List floating-ip availables
```
nova floating-ip-list
```

Assign a floating-ip to this instance.
```
nova add-floating-ip cirrOS 172.30.242.152
```

## Connect to your instance
First, try to ping your instance
```
ping 172.30.242.152
```

Now, connect in this instance by ssh
```
ssh cirros@172.30.242.152
```
