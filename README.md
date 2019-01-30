# Ansible Cloud Demos
This repository contains a collection of Ansible playbooks that are designed to help demo provisioning and orchestration in public clouds like AWS, Azure and GCP. Virtualization platforms like Red Hat Virtualization (RHV) are also included.
# Use of Python Virtual Environments
Many Ansible cloud modules require the use of extra Python packages. To avoid clutter, we will use Python virtual environments for each of the demos. More info on Python virtual environments can be found in the Python documentation [1], but essentially they are used “to manage Python packages for different projects. Using virtualenv allows you to avoid installing Python packages globally which could break system tools or other projects.” Using virtual environments is also a convenient way to practice setting up Python for use with various Ansible modules.
## Setting Up a Python Virtual Environment
I usually create virtual environment directories in ~/python. To create a virtual environment for our demo, cd to ~/python and run the following command:
```bash
virtualenv demo-27
```
The name demo-27 could be anything, but here it represents a demo environment using the default Python 2.7 base.

To activate the newly created Python virtual environment, run the following command:
```bash
source ~/python/demo-27/bin/activate
```
If successful, your terminal prompt should be prefixed w/ “(demo-27)”.
# External Links
[1] - https://packaging.python.org/guides/installing-using-pip-and-virtualenv/
