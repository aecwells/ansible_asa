# Ansible Blue-Green Deployment with Cisco ASA


## Overview

This repository contains an Ansible playbook and inventory file designed for performing a blue-green deployment on multiple domains using Cisco ASA for network configuration. The playbook switches the active environment for each domain and verifies the deployment by checking HTTP and HTTPS endpoints.

## Inventory File

The inventory file (`multi_domain_blue_green_deployment.yml`) defines the hosts, variables, and domain configurations for the playbook.

### Inventory File structure

```yaml
all:
  vars:
    asa_host: "asa.example.com"  # The IP or hostname of your ASA device
    asa_user: "admin"  # Your ASA username
    asa_ssh_key: "/path/to/your/ssh/key"  # Path to your SSH key
    domains:
      - name: "example1.com"
        current_active_env: "blue"
        blue_vms:
          - "192.168.1.101"
          - "192.168.1.102"
        green_vms:
          - "192.168.1.201"
          - "192.168.1.202"
      - name: "example2.com"
        current_active_env: "green"
        blue_vms:
          - "192.168.2.101"
          - "192.168.2.102"
        green_vms:
          - "192.168.2.201"
          - "192.168.2.202"
  children:
    localhost:
      hosts:
        localhost
```

### Variables

* `asa_host`: The IP or hostname of your ASA device.
* `asa_user`: The username for ASA login.
* `asa_ssh_key`: The path to your SSH key for ASA authentication.
* `domains`: A list of domain configurations. Each domain has:
  * `name`: The domain name.
  * `current_active_env`: The currently active environment (blue or green).
  * `blue_vms`: A list of VM IPs for the blue environment.
  * `green_vms`: A list of VM IPs for the green environment.

## Playbook

The playbook (`multi_domain_blue_green_deployment.yml`) performs the following tasks for each domain:

1. Determines the inactive environment.
2. Sets variables for the inactive environment.
3. Prints the current and new environment details.
4. Updates the ASA configuration to switch to the inactive environment.
5. Verifies the new environment is serving traffic on HTTP and HTTPS.
6. Updates the active environment to the newly activated environment.
7. Prints a completion message.

### Playbook Structure

```yaml
---
- name: Blue-Green Deployment with Cisco ASA for Multiple Domains
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Iterate over each domain
      loop: "{{ domains }}"
      vars:
        domain: "{{ item.name }}"
        sanitized_domain: "{{ domain | replace('.', '_') }}"
        current_active_env: "{{ item.current_active_env }}"
        blue_vms: "{{ item.blue_vms }}"
        green_vms: "{{ item.green_vms }}"
      tasks:
        - name: Determine the inactive environment for {{ domain }}
          set_fact:
            inactive_env: "{{ 'green' if current_active_env == 'blue' else 'blue' }}"

        - name: Set inactive environment variables for {{ domain }}
          set_fact:
            inactive_vms: "{{ green_vms if inactive_env == 'green' else blue_vms }}"

        - name: Print current active environment for {{ domain }}
          debug:
            msg: "Domain: {{ domain }} - Current active environment: {{ current_active_env }}"

        - name: Print new environment to activate for {{ domain }}
          debug:
            msg: "Domain: {{ domain }} - Switching to environment: {{ inactive_env }}"

        - name: Update ASA configuration to use the new environment for {{ domain }}
          cisco.asa.asa_config:
            lines:
              - "object network {{ sanitized_domain }}_{{ inactive_env }}_http"
              - "host {{ item }}"
              - "nat (inside,outside) static interface service tcp 80 80"
              - "object network {{ sanitized_domain }}_{{ inactive_env }}_https"
              - "host {{ item }}"
              - "nat (inside,outside) static interface service tcp 443 443"
            with_items: "{{ inactive_vms }}"
            provider:
              host: "{{ asa_host }}"
              username: "{{ asa_user }}"
              ssh_keyfile: "{{ asa_ssh_key }}"
              authorize: yes
            save_config: yes

        - name: Verify the new environment is serving traffic for {{ domain }} on HTTP (port 80)
          uri:
            url: "http://{{ domain }}"
            status_code: 200
          register: http_result
          until: http_result is succeeded or http_result.status == 301
          retries: 5
          delay: 10

        - name: Verify the new environment is serving traffic for {{ domain }} on HTTPS (port 443)
          uri:
            url: "https://{{ domain }}"
            status_code: 200
          register: https_result
          until: https_result is succeeded
          retries: 5
          delay: 10

        - name: Update current active environment for {{ domain }}
          set_fact:
            item.current_active_env: "{{ inactive_env }}"

        - name: Print completion message for {{ domain }}
          debug:
            msg: |
              Domain: {{ domain }}
              - Deployment completed.
              - Current active environment: {{ item.current_active_env }}
              - HTTP (port 80) verification result: {{ http_result }}
              - HTTPS (port 443) verification result: {{ https_result }}
```

### Usage

1. Prepare the Inventory File: Update inventory.yml with your ASA device details, SSH key path, and domain configurations.
2. Run the Playbook: Execute the playbook using the following command:
```bash
ansible-playbook -i inventory.yml playbook.yml
```
3. Verify Deployment: The playbook will automatically verify that the new environment is serving traffic correctly on both HTTP and HTTPS.

## Requirements
* Ansible 2.9 or later
* Cisco ASA device with SSH access and appropriate configuration permissions
* Python cisco.asa collection for ASA configurations

## Notes
* Ensure the SSH key has appropriate permissions and access to the ASA device.
* The playbook includes retries for HTTP and HTTPS checks to handle any initial delays in traffic switchover.

## Troubleshooting

* **Authentication Errors**: Check the `asa_user` and `asa_ssh_key` paths.
* **Configuration Errors**: Ensure the ASA device has the necessary configuration permissions for the commands used.
* **Traffic Verification Failures**: Verify network connectivity and that the VMs are correctly serving the required services.
