# **Hashicorp Consul by Ansible**

## **Description**

Install and configure [Hashicorp Consul](https://www.consul.io/)

- **Consul Cluset in High Availability Mode**
- **Secure Consul Agent Communication with Encryption and Certificates**
- **Consul UI for HTTPS**

## **Requirements**

- **Ansible**

  - Python 2.7
  - ansible 2.9+

- **Servers**

  - 2 disks on the servers (for consul data)

## **Pre Tasks**

### **SSH**

- **Create a key pair ssh to connect on hosts:**

  ```shell
  cd ./keys
  sh ssh-keygen.sh <name your key pair>
  ```

- **Copy the public key to the remote hosts:**

  ```shell
  ssh-copy-id -i <your public ssh key> user@<host>
  ```

### **Inventory**

- **Create a new inventory:**

  ```shell
  cp -rf ./inventories/sample ./inventories/<inventory name>
  ```

- **Configure the hosts in the inventory:**

  ```shell
  vim ./inventories/<inventory name>/hosts.yml
  ```

  Ex.

  ```yaml
  ---
  all:
  vars:
  hosts:
      consul01.lab.local:
      ansible_host: 172.16.252.101
      vault01.lab.local:
      ansible_host: 172.16.252.111
  children:
      server:
      hosts:
          consul01.lab.local:
      client:
      hosts:
          vault01.lab.local:
  ```

### **Custom**

- **Configure common variables:**

  ```shell
  vim ./roles/common/vars/main.yml
  ```

  ```yaml
  ---
  packages:
  to_install:
      - epel-release
      - wget
      - curl

  services:
  to_enable:
      - network

  to_disable:
      - firewalld

  ntp_servers:
  - "server 0.br.pool.ntp.org"
  - "server 1.br.pool.ntp.org"
  ```

- **Configure the variables for openssl:**

  ```shell
  vim ./roles/openssl/vars/main.yml
  ```

  ```yaml
  # All
  cert_path: "/consul/data/certificates/ssl"

  # CA Configuration
  key_size: 4096
  type_algorithm: "RSA"
  secret_ca_passphrase: "555f9ea62f39918d069c2590a38e49f22af58af2483b156c8b7eeface666b410"
  ca_file_key: "ca.key"
  ca_file_certificate: "ca.pem"
  common_name: "Consul CA"

  # Server Configuration
  server_file_key: "server.key"
  server_file_certificate: "server.pem"
  server_subject_alt_name:
    domain:
      - "DNS:*.lab.local"
      - "DNS:server.dc1.consul"
      - "DNS:{{ ansible_hostname }}"

  # Client Configuration
  client_file_key: "client.key"
  client_file_certificate: "client.pem"
  client_subject_alt_name:
    domain:
      - "DNS:*.lab.local"
      - "DNS:client.dc1.consul"
      - "DNS:{{ ansible_hostname }}"
  ```

- **Configure the variables for server hosts:**

  ```shell
  vim ./roles/server/vars/main.yml
  ```

  ```yaml
  ---
  verify_incoming: "false"
  verify_outgoing: "true"
  verify_server_hostname: "true"
  enable_script_checks: "false"
  disable_remote_exec: "true"
  path_certificate: "/consul/data/certificates/ssl"
  ca_file: "{{ path_certificate }}/ca.pem"
  cert_file: "{{ path_certificate }}/server.pem"
  key_file: "{{ path_certificate }}/server.key"
  server: "true"
  datacenter: "dc1"
  data_dir: "/consul/data"
  bind_addr: "0.0.0.0"
  client_addr: "0.0.0.0"
  bootstrap_expect: 3
  retry_join:
  to_join:
      - "{{ groups['server'][0] }}"
      - "{{ groups['server'][1] }}"
      - "{{ groups['server'][2] }}"
  ui_config_enabled: "true"
  ui_http_port: "8500"
  ui_https_port: "8501"
  log_level: "DEBUG"
  enable_syslog: "true"
  ```

- **Configure the variables for client hosts:**

  ```shell
  vim ./roles/client/vars/main.yml
  ```

  ```yaml
  ---
  verify_incoming: "true"
  verify_outgoing: "true"
  path_certificate: "/etc/consul.d/certificates/ssl"
  tmp_path_certificates: "/consul/data/certificates/ssl"
  data_dir: "/etc/consul.d"
  ca_file: "{{ path_certificate }}/ca.pem"
  cert_file: "{{ path_certificate }}/client.pem"
  key_file: "{{ path_certificate }}/client.key"
  server: "false"
  datacenter: "dc1"
  bind_addr: "{{ ansible_host }}"
  client_addr: "127.0.0.1"
  bootstrap_expect: 3
  retry_join:
    to_join:
      - "{{ groups['all'][0] }}"
      - "{{ groups['all'][1] }}"
      - "{{ groups['all'][2] }}"
  log_level: "DEBUG"
  enable_syslog: "true"

  # Client Configuration
  ca_file_certificate: "ca.pem"
  client_file_key: "client.key"
  client_file_certificate: "client.pem"
  client_subject_alt_name:
    domain:
      - "DNS:*.lab.local"
      - "DNS:client.dc1.consul"
      - "DNS:{{ ansible_hostname }}"
  ```

## **Run Playbook**

- ### **Server**

  ```shell
  ansible-playbook -i inventories/<inventory name>/hosts.yml server.yml
  ```

- ### **Client**

  ```shell
  ansible-playbook -i inventories/<inventory name>/hosts.yml client.yml
  ```
