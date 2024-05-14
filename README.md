# Server Provisioning Guide

This guide outlines the process for provisioning a server through automated methods using Ansible. Ensure you have met all the prerequisites before beginning the server provisioning.

## Prerequisites

Before you begin the server provisioning process, please ensure the following prerequisites are met:

-  **SSH Access**: Proper SSH access is required for Ansible to connect to the host machine.
-  **Passwordless SSH**: Setup SSH keys to allow passwordless access to the server, which is crucial for Ansible automation scripts to run smoothly.
- On first run, use `--ask-become-pass` for the root password.

## Create Inventory File

Ansible utilizes an inventory file to track and manage the servers. Below is an example of how to create an inventory file for your environment. This file should list all hosts under their respective groups and define essential variables needed for the provisioning script.

```ini
[hass]
342.234.342.24

[hass:vars]
ansible_ssh_port=4328
security_ssh_port=4328
ansible_user=gwbrck
user_passwordstore_entry=server
github_name=gwbrck
tunnel=false
```

### Description of Variables

-  **ansible_ssh_port and security_ssh_port**: These variables define the SSH port; ensure that the server's firewall configuration allows connections on this port.
-  **ansible_user**: The username used by Ansible to log into the host machines.
-  **user_passwordstore_entry**: Reference to the password storage entry. This should be secured and only be accessible by the user that runs the Ansible.
-  **github_name**: Your GitHub username for configurations that involve pulling resources from your GitHub repositories.
-  **tunnel**: A boolean flag (true/false) to configure whether to use a tunnel for connection.

## Running the Provisioning Script

Once your inventory file is set up, you can run the provisioning script by executing the following command:
```bash
ansible-playbook run.yaml
```

For further assistance or more information, refer to the [official Ansible documentation](https://docs.ansible.com/).
