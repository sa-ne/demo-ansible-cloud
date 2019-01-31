# Configure Ansible to Work with Azure

Before we start working with the Anisble Azure modules we need to do two things. First, the Ansible Azure modules depend on some Python packages (like the Azure SDK) so we will need to install them. Second, we need to configure credentials so the Ansible modules can access resources in Azure.

## Installing Python Packages

First let's source our Python virtual environment (see `../README.md` for more information).

```bash
source ~/python/demo-27/bin/activate
```

Now we can install the required Python package dependencies in our virtual environment.

```bash
pip install ansible[azure]
```

For good measure, source our virtual environment again.

```bash
source ~/python/demo-27/bin/activate
```

## Configuring Credentials

We will need to create a service principal to allow Ansible to access resources in Azure. More info on this can be found on the Microsoft website [1]. Once you have your subscription id, tenant id, client id and secret you can simply store these values in the file `~/.azure/credentials`. The credentials file is an ini style file, an example is shown below.

```ini
[default]
subscription_id=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
client_id=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
secret=xxxxxxxxxxxxxxxxx
tenant=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

More information on configuring Ansible to work w/ Azure can be found on the Ansible website [2].

# Ansible Azure Demos

The following demos are available:

* Provision LAMP Stack
* Rolling Patch/Reboot of LAMP Stack
* Deprovision Environment

## Prerequisites
### Configuring a Vault

Before running our playbooks we need to create an Ansible vault with the following variables.

|Variable Name|Description|Example|
|:---|:---|:---|
|app\_database\_username|Username for MySQL Instance|app_admin|
|app\_database\_password|Password for MySQL Instance|$3Cr3t|
|app\_server\_username|Username to access VMs over SSH|azure-user|
|app\_server\_public\_key|Public key for our user|ssh-rsa ... user@azure|

For reference, to create a vault run the following command:

```bash
ansible-vault create vault.yml
```

Vault contents example:

```yaml
---
app_database_username: "app_admin"
app_database_password: "$3Cr3t"
app_server_username: "azure-user"
app_server_public_key: "ssh-rsa ... user@azure"
```

### Creating an Inventory File

To support the installation in Azure, we need to create an inventory file with the appropriate variables. The inventory will also contain a list of VMs that will be created to support the application.

|Variable Name|Description|Example|
|:---|:---|:---|
|app\_prefix|Used to prefix resources in Azure for unique deployments|test|
|app\_source|Git repository for LAMP application|https://github.com/sa-ne/simple-lamp-application.git|
|azure\_location|Azure region to deploy to|eastus|
|azure\_cidr\_network|Address space (in CIDR notation) for Azure Virtual Network|10.254.252.0/22|
|azure\_app\_subnet|Address space (in CIDR notation) for a subnet that will be defined in the Azure Virtual Network|10.254.252.0/24|
|azure\_infra\_resource\_group| (Optional) If you would like to integrate with a DNZ zone in an existing resource group, name that resource group here|my-infra-rg-eastus|
|azure\_infra\_zone| (Optional) DNS zone to use for DNS A records|azure.redhat-demo.com|

You will also need to define a list of VMs to provision in the 'web' group. If you want to perform the rolling reboot demo, a minimum of two VMs should be defined.

An example of a working inventory file is included below:

```yaml
---
all:
  vars:
    app_prefix: "test"
    app_source: "https://github.com/sa-ne/simple-lamp-application.git"
    azure_location: eastus
    azure_cidr_network: 10.254.252.0/22
    azure_app_subnet: 10.254.252.0/24
    azure_infra_resource_group: "my-infra-rg-eastus"
    azure_infra_zone: "azure.redhat-demo.com"
  children:
    web:
      hosts:
        app-vm1:
        app-vm2:
        app-vm3:
```

## Provisioning LAMP Stack

This demo is broken out into three playbooks. They will accomplish the following:

|Playbook Name|Description|
|:---|:---|
|1-create-resource-group-misc.yml|Creates a resource group, virtual network, subnet and security group for our application.|
|2-create-database.yml|Creates an instance of MySQL in the Azure Database for MySQL Service, creates a proper MySQL database in the instance and then updates firewall rules to allow VMs to connect to the database.|
|3-create-vms.yml|Creates VMs (defined in inventory), a load balancer, public IPs, NICs, DNS entries (optional) and a VM availability set. Will also configure the VMs (install packages and deploy the application).|

Run the first playbook as follows (we only need the inventory file here):

```bash
ansible-playbook -i inventory.yml 1-create-resource-group-misc.yml
```

The second playbook will also need access to the vault, run it as follows:

```bash
ansible-playbook -i inventory.yml --ask-vault-pass -e @vault.yml 2-create-database.yml
```

Finally, the third playbook will actually configure hosts so we will need to pass in credentials for the VMs. Run it as follows:

```bash
ansible-playbook -i inventory.yml --ask-vault-pass -e @vault.yml -u azure-user --key-file ~/.ssh/my_private_key 3-create-vms.yml
```

If you do not have access to a DNS zone, you can have the third playbook skip over the relevant DNS tasks by adding the `--skip-tags dns` flag to the `ansible-playbook` command.

```bash
ansible-playbook -i inventory.yml --ask-vault-pass -e @vault.yml --skip-tags dns -u azure-user --key-file ~/.ssh/my_private_key 3-create-vms.yml
```

## Rolling Patch/Reboot of LAMP Stack

After provisioning your LAMP stack, you can perform the rolling patch/reboot demo as follows:

```bash
ansible-playbook -i inventory.yml -u azure-user --key-file ~/.ssh/my_private_key rolling-patch-demo.yml
```

The application should remain available via the load balancer while the VMs are being rebooted in serial. To do this, the playbook will remove the HTTP monitor from the system and wait the appropriate amount of time. Next it will stop services, patch the VM, reboot the VM, wait for the VM to come back on-line, enable services, restore the monitor and then proceed to the next system.

## Deprovision Environment

This playbook will remove all the resources we created (i.e. delete resource group and DNS entries, if applicable).

To deprovision your environment, run the `delete-environment.yml` playbook as follows:

```bash
ansible-playbook -i inventory.yml delete-environment.yml
```

If you did not have access to a DNS zone when running the original demo, you can skip over the relevant DNS tasks by adding the `--skip-tags dns` flag to the `ansible-playbook` command.

```bash
ansible-playbook -i inventory.yml --skip-tags dns delete-environment.yml
```

# External Links

[1] - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal </br>
[2] - https://docs.ansible.com/ansible/latest/scenario_guides/guide_azure.html
