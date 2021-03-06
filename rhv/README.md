# Configure Ansible to Work with oVirt/RHV

Before we start working with the Ansible oVirt modules we need to install some Python packages (like the oVirt SDK).

## Installing Python Packages

First let's source our Python virtual environment (see `../README.md` for more information).

```bash
source ~/python/demo-27/bin/activate
```

Now we can install the required Python package dependencies in our virtual environment.

```bash
pip install ovirt-ansible ovirt-engine-sdk-python
```

The roles contained in this demo also require the use of `dnspython` and `netaddr` packages.

```bash
pip install dnspython netaddr
```

For good measure, source our virtual environment again.

```bash
source ~/python/demo-27/bin/activate
```

# Ansible oVirt Demos

The following demos are available:

* Provision n Number of Virtual Machines (The Provisioner) Integrated with IPAM, IdM and Satellite
* Configure Virtual Machines for OpenShift 3.11
* Deprovision n Number of Virtual Machines Integrated with IPAM, IdM and Satellite

## About The Provisioner

The provisioner is a playbook that executes a series of roles and tasks to deploy n number of virtual machines with various configurations on oVirt/RHV. It consists of the following roles:

|Role|Description|
|:---|:---|
|dynammic-inventory|Generates a dynamic in-memory inventory used to affect changes on virtual machines we are provisioning. Hosts are placed in the `dynamic_inventory` group.|
|ipa|Creates host accounts in IdM and assigns random one time passwords to these accounts.|
|ipa-retire|Deletes previously provisioned host accounts in IdM|
|ipam-inventory|Populates `network_name_dict`<sup>1</sup> based on variables defined in inventory file.|
|ipam-phpipam|Populates `network_name_dict`<sup>1</sup> by requesting IPs and other network attributes from phpIPAM|
|ipam-phpipam-retire|Frees up previously allocated IPs in phpIPAM|
|ipa-openshift|Creates specific DNS entries to support an OpenShift installation|
|ipa-openshift-retire|Removes previous created DNS entries|
|openshift-prep|Prepares a node for the installation of OpenShift 3.11|
|rhv|Provisions virtual machines in oVirt/RHV.|
|rhv-retire|Removes virtual machines in oVirt/RHV|
|satellite|Registers virtual machines to Satellite|
|satellite-retire|Unregisters virtual machines from Satellite|

Note <sup>1</sup>: The variable `network_name_dict` is used to define network information for all the virtual machines provisioned. This way any other IPAM solution could be used as long the user develops a role to set the variable.

An example of how the IPAM roles could configure `network_name_dict`:

```yaml
network_name_dict:
  vm1.ose.local:
    gateway: 172.16.10.254
    ip: 172.16.10.101
    mask: 255.255.255.0
    nameservers: 172.16.10.11
  vm2.ose.local:
    gateway: 172.16.10.254
    ip: 172.16.10.102
    mask: 255.255.255.0
    nameservers: 172.16.10.11
```

### Notes on IdM

To make use of the IdM integration, you are required to have DNS management enabled and a zone supporting the DNS suffix for your environment (i.e. search parameter in your inventory file).

### Notes on Satellite

When provisioning OpenShift instances, make sure your activation key contains the appropriate repositories [1] per documentation.

### Notes on phpIPAM

When using phpIPAM, the host variable `ipam_cidr` is used. This CIDR block must be defined as a subnet in phpIPAM. This subnet must also be configured with the default gateway and DNS. I have created a role to assist with the installation of phpIPAM on RHEL 7 [2]. Gateways for subnets are defined by checking the appropriate checkbox on the applicable IP properties within the subnet. DNS configuration is handled in the actual subnet properties. You will also need to enable the API in the phpIPAM settings and create an API id.

## Prerequisites
### Configuring a Vault

Before running our playbooks we need to create an Ansible vault with the following variables:

|Variable Name|Description|Example|
|:---|:---|:---|
|root_password|Root Password for Virtual Machines|$3cr3t|
|rhv_hostname|RHV Manager Hostname|rhevm.local|
|rhv_username|Administrator username for RHV Manager|admin@internal|
|rhv_password|Administrator password for RHV Manager|$3cr3t|
|rhv_cluster|RHV Cluster to Deploy Virtual Machines|my\_cluster|
|ipa_hostname|IdM Hostname|idm1.local|
|ipa_username|Administrator username for IdM|admin|
|ipa_password|Administrator password for IdM|$3cr3t|
|satellite_hostname|Satellite Hostname|sputnik.local|
|satellite_username|Administrator username for Satellite|admin|
|satellite_password|Administrator password for Satellite|$3cr3t|
|satellite\_org\_id|Satellite Organization Name|My_Organization|
|phpipam_hostname|phpIPAM Hostname|ipam.local|
|phpipam_username|Administrator username for phpIPAM|admin|
|phpipam_password|Administrator password for phpIPAM|$3cr3t|
|phpipam_appid|API Application ID for phpIPAM|my\_org\_api|

For reference, to create a vault run the following command:

```bash
ansible-vault create vault.yml
```

Vault contents example:

```yaml
---
root_password: $3cr3t
rhv_hostname: rhevm.local
rhv_username: admin@internal
rhv_password: $3cr3t
rhv_cluster: my_cluster
ipa_hostname: idm1.local
ipa_username: admin
ipa_password: $3cr3t
satellite_hostname: sputnik.local
satellite_username: admin
satellite_password: $3cr3t
satellite_org_id: My_Organization
phpipam_hostname: ipam.local
phpipam_username: admin
phpipam_password: $3cr3t
phpipam_appid: my_org_api
```

### Creating an Inventory File

To support the installation of virtual machines on oVirt/RHV, we need to create an inventory file with the appropriate variables as a list of VMs that will be created.

#### Global Variables

|Variable Name|Description|Example|
|:---|:---|:---|
|provision\_group|Group in inventory that will be provisioned.|my\_vms|
|ipam\_phpipam|If True, use the ipam-phpipam role to acquire IPs and other network information (gateway, mask, DNS).|True/False|
|ipam\_inventory|If True, use the ipam-inventory role to define IPs and other network information (gateway, mask, DNS).|True/False|
|satellite\_activation\_key|Activation key used to register virtual machines to Satellite|"My Activation Key"|
|cleanup\_known\_hosts|Useful for iterative provisioning. Will remove references to hostnames/IPs in known\_hosts file based off inventory|True/False|

#### Host Variables

|Variable Name|Description|Example|
|:---|:---|:---|
|ipam_cidr|Only used when `ipam_phpipam` is set to True. Define the CIDR block used to request the next available IP from phpIPAM|172.16.10.0/24|
|ip|Only used when `ipam_inventory` is set to True. IP address of the virtual machine.|172.16.10.101|
|mask|Only used when `ipam_inventory` is set to True. The subnet associated with the IP of the virtual machine.|255.255.255.0|
|gateway|Only used when `ipam_inventory` is set to True. The gateway associated with teh IP of the virtual machine.|172.16.10.254|
|nameservers|Only used when `ipam_inventory` is set to True. The nameservers used for DNS resolution.|
|search|The DNS search suffix.|ose.local|
|memory|The amount of RAM for the virtual machine.|16GiB|
|sockets|The number of sockets for the virtual machine.|1|
|cores|The number of cores for each socket of the virtual machine.|4|
|template|The name of the template defined in oVirt/RHV that the VM will use.|rhel-server-7.6-x86_64-kvm-60G|
|storage_domain|The storage domain in oVirt/RHV that will hold the VM root disk.|nfs-vms|
|disks|An optional list of additional disks to add to the virtual machine|See disks below|

###### Disks Variable

The disk variable contains a list of attributes for each additional disk with the following attributes:

|Variable Name|Description|Example|
|:---|:---|:---|
|name|The name of the disk.|docker-disk|
|size|The size of the disk.|40GiB|
|format|The format of the disk.|cow|
|interface|The interface of the disk.|virtio_scsi|
|storage_domain|The storage domain that will hold the disk.|nfs-vms|

#### Example Inventory File (ipam-inventory)

An example of a working inventory file for use with the ipam-inventory role is included below (note the use of ip, gateway, mask and nameserver variables):

```yaml
---
all:
  vars:
    provision_group: my_vms
    ipam_phpipam: False
    ipam_inventory: True
    satellite_activation_key: "My Activation Key"
  children:
    my_vms:
      hosts:
        vm1.ose.local:
          ip: 172.16.10.101
          gateway: 172.16.10.254
          mask: 255.255.255.0
          nameservers: 172.16.10.11
          search: ose.local
          memory: 16GiB
          sockets: 1
          cores: 4
          template: rhel-server-7.6-x86_64-kvm-60G
          storage_domain: nfs-vms
          disks:
          - name: docker-disk
            size: 40GiB
            format: cow
            interface: virtio_scsi
            storage_domain: nfs-vms
        vm2.ose.local:
          ip: 172.16.10.102
          gateway: 172.16.10.254
          mask: 255.255.255.0
          nameservers: 172.16.10.11
          search: ose.local
          memory: 16GiB
          sockets: 1
          cores: 4
          template: rhel-server-7.6-x86_64-kvm-60G
          storage_domain: nfs-vms
          disks:
          - name: docker-disk
            size: 40GiB
            format: cow
            interface: virtio_scsi
            storage_domain: nfs-vms
```

#### Example Inventory File (ipam-phpipam)

An example of a working inventory file for use with the ipam-phpipam role is included below (ipam_cidr is used instead of ip, mask, gateway and nameserver variables):

```yaml
---
all:
  vars:
    provision_group: my_vms
    ipam_phpipam: False
    ipam_inventory: True
    satellite_activation_key: "My Activation Key"
  children:
    my_vms:
      hosts:
        vm1.ose.local:
          ipam_cidr: 172.16.10.0/24
          search: ose.local
          memory: 16GiB
          sockets: 1
          cores: 4
          template: rhel-server-7.6-x86_64-kvm-60G
          storage_domain: nfs-vms
          disks:
          - name: docker-disk
            size: 40GiB
            format: cow
            interface: virtio_scsi
            storage_domain: nfs-vms
        vm2.ose.local:
          ipam_cidr: 172.16.10.0/24
          search: ose.local
          memory: 16GiB
          sockets: 1
          cores: 4
          template: rhel-server-7.6-x86_64-kvm-60G
          storage_domain: nfs-vms
          disks:
          - name: docker-disk
            size: 40GiB
            format: cow
            interface: virtio_scsi
            storage_domain: nfs-vms
```

## Provisioning n Number of Virtual Machines

This demo will allow you to provison a series of virtual machines in RHV that are integrated with phpIPAM (optional), IdM and Satellite. To run the demo, simply configure your inventory file and vault as noted above and run the following command:

`ansible-playbook --ask-vault-pass -u root -k -i inventory provision.yml`

## Preparing Provisioned Virtual Machines for OpenShift

This demo will allow you to configure the previously provisioned nodes for OpenShift 3.11.

Some notes about this demo:

* The hostname pattern for masters must be \*masterN\*.\* (i.e. master1.ose.local)
* The hostname pattern for infrastructure nodes must be \*infra-nodeN\*.\* (i.e. infra-node1.ose.local)
* Optionally, the hostname pattern for application nodes should be \*app-nodeN\*.\* (i.e. app-node1.ose.local)

To prepare your previously provisioned nodes for OpenShift, run the following command:

`ansible-playbook --ask-vault-pass -u root -k -i inventory openshift-prep.yml`

## Retiring Virtual Machines

This demo will free up IPs allocated in phpIPAM (optional), remove hosts from IdM and Satellite and delete the virtual machines in RHV. Run the following command to retire:

`ansible-playbook --ask-vault-pass -i inventory.yml retire.yml`

## Cleaning up OpenShift Artifacts

As part of the OpenShift prep, a few DNS entries were created in IdM. To remove those entries, run the following:

`ansible-playbook --ask-vault-pass -i inventory openshift-retire.yml`

# External Links
[1] - https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html-single/installing_clusters/#host-registration </br>
[2] - https://github.com/nasx/phpipam-install
