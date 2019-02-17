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
