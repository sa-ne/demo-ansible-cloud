# Configure Ansible to Work with AWS

Before we start working with the Anisble AWS modules we need to do two things. First, the Ansible AWS modules depend on some Python packages (like the AWS SDK) so we will need to install them. Second, we need to configure credentials so the Ansible modules can access resources in AWS.

## Installing Python Packages

First let's source our Python virtual environment (see `../README.md` for more information).

```bash
source ~/python/demo-27/bin/activate
```

Now we can install the required Python package dependencies in our virtual environment.

```bash
pip install boto boto3
```

For good measure, source our virtual environment again.

```bash
source ~/python/demo-27/bin/activate
```

## Configuring Credentials

To access the AWS API we will need to setup Access Keys in for a user in IAM. More info on this can be found on the Amazon website [1]. Once you have your access key id and secret access key you can simply store these vaule sin the file `~/.aws/credentials`. The credentials file is an ini style file, an example is shown below.

```ini
[default]
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
aws_access_key_id = XXXXXXXXXXXXXXXXXXXX
```

More information on configuring Ansible to work w/ AWS can be found on the Ansible website [2].

# Ansible AWS Demos

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

For reference, to create a vault run the following command:

```bash
ansible-vault create vault.yml
```

Vault contents example:

```yaml
---
app_database_username: "app_admin"
app_database_password: "$3Cr3t"
```

### Creating an Inventory File

To support the installation in AWS, we need to create an inventory file with the appropriate variables. The inventory will also contain a list of VMs that will be created to support the application.

|Variable Name|Description|Example|
|:---|:---|:---|
|app\_prefix|Used to prefix resources in AWS for unique deployments|test|
|app\_database\_name|Name of database inside the MySQL instance|appdb|
|app\_source|Git repository for LAMP application|https://github.com/sa-ne/simple-lamp-application.git|
|aws\_region|AWS region<sup>1</sup> to deploy to|us-east-1|
|aws\_infra\_zone|DNS zone to use for DNS a records|aws.redhat-demo.com|
|aws\_database\_engine|Type of database|mariadb|
|aws\_database\_instance\_type|Database instance size|db.t2.small|
|aws\_database\_size|Database instance storage (in GB)|5|
|aws\_database\_license\_model|The license model for the DB instance|general-public-license|
|aws\_database\_multi\_zone|Specifies if this is a multi-availability-zone deployment.|yes|
|aws\_ami\_rhel\_version|The version of RHEL you want to use|7.6|
|aws\_key\_name|SSH public key (documentation here [3])|my_key|
|aws\_instance\_type|EC2 instance size|t2.small|
|aws\_redhat\_owner\_id|Red Hat AMI owner id|309956199498|

Note<sup>1</sup>: If you deploy to a region other than us-east-1, you must create the appropriate YAML for that regions subnets in `subnets/`.

You will also need to define a list of VMs to provision in the 'web' group. If you want to perform the rolling reboot demo, a minimum of two VMs should be defined.

An example of a owkring inventory file is included below:

```yaml
---
all:
  vars:
    app_prefix: ckeller-test
    app_database_name: ckellertest
    app_source: https://github.com/sa-ne/simple-lamp-application.git
    aws_region: us-east-1
    aws_infra_zone: aws.redhat-demo.com
    aws_database_engine: mariadb
    aws_database_instance_type: db.t2.small
    aws_database_size: 20
    aws_database_license_model: general-public-license
    aws_database_multi_zone: no
    aws_ami_rhel_version: 7.6
    aws_key_name: my_key
    aws_instance_type: t2.small
    aws_redhat_owner_id: 309956199498
  children:
    web:
      hosts:
        app-vm1:
          subnet: "us-east-1a"
        app-vm2:
          subnet: "us-east-1b"
        app-vm3:
          subnet: "us-east-1c"
```

## Provisioning LAMP Stack

This demo is broken out into three playbooks. They will accomplish the following:

|Playbook Name|Description|
|:---|:---|
|1-create-vpc-misc.yml|Creates a VPC, subnets, IGWs and security groups for our application.|
|2-create-database.yml|Creates an instance of RDS for our application.|
|3-create-vms.yml|Creates VMs (defined in inventory), a load balancer, DNS entries (optional) and a VMs. Will also configure the VMs (install packages and deploy the application).|

Run the first playbook as follows (we only need the inventory file here):

```bash
ansible-playbook -i inventory.yml 1-create-vpc-misc.yml
```

The second playbook will also need access to the vault, run it as follows:

```bash
ansible-playbook -i inventory.yml --ask-vault-pass -e @vault.yml 2-create-database.yml
```

Finally, the third playbook will actually configure hosts so we will need to pass in credentials for the VMs. Run it as follows:

```bash
ansible-playbook -i inventory.yml -u ec2-user --key-file ~/.ssh/my_private_key --ask-vault-pass -e @vault.yml 3-create-vms.yml
```

If you do not have access to a DNS zone, you can have the third playbook skip over the relevant DNS tasks by adding the `---skip-tags dns` flag to the `ansible-playbook` command. Please note that the aws\_infra\_zone variablea must still be defined, however it can be set to a dummy value (i.e. test.local).

```bash
ansible-playbook -i inventory.yml --skip-tags dns -u ec2-user --key-file ~/.ssh/my_private_key --ask-vault-pass -e @vault.yml 3-create-vms.yml
```

## Rolling Patch/Reboot of LAMP Stack

After provisioning your LAMP stack, you can perform the rolling patch/reboot demo as follows:

```bash
ansible-playbook -i inventory.yml -u ec2-user --key-file ~/.ssh/my_private_key rolling-patch-demo.yml
```

The application should remain available via the load balancer while the VMs are being rebooted in serial. To do this, the playbook will remove the HTTP monitor from the system and wait the appropriate amount of time. Next it will stop services, patch the VM, reboot the VM, wait for the VM to come back on-line, enable services, restore the monitor and then proceed to the next system.

## Deprovision Environment

This playbook will remove all the resources we created.

To deprovision your environment, run the `delete-environment.yml` playbook as follows:

```bash
ansible-playbook -i inventory.yml delete-environment.yml
```

If you did not have access to a DNS zone when running the original demo, you can skip over the relevant DNS tasks by adding the `--skip-tags dns` flag to the `ansible-playbook` command.

```bash
ansible-playbook -i inventory.yml --skip-tags dns delete-environment.yml
```

# External Links

[1] - https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html <br />
[2] - https://docs.ansible.com/ansible/latest/scenario_guides/guide_aws.html <br />
[3] - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
