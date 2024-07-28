Here's a detailed `README.md` file that explains how to use the Ansible playbook to deploy and configure the NESTJS boilerplate on a remote Ubuntu 22.04 server:

---

# Automated Deployment and Configuration with Ansible for NESTJS Boilerplate

This repository contains an Ansible playbook that automates the deployment and configuration of the NESTJS boilerplate on a remote Ubuntu 22.04 server.

## Prerequisites

1. **Ansible**: Install Ansible on your local machine.

   ```sh
   sudo apt update
   sudo apt install ansible
   ```

2. **SSH Access**: Ensure SSH access to your remote server with sudo privileges.

3. **Environment Variables**: Create a `.env` file on your local machine with the necessary sensitive values.

## Setup

### 1. Create the `.env` File

Create a `.env` file in the same directory as your playbook with the following content:

```env
REPO_URL=https://github.com/hngprojects/hng_boilerplate_nestjs.git
BRANCH=devops
APP_DIR=/opt/stage_5b
PG_PASSWORD=your_pg_password
DB_NAME=mydatabase
DB_USER=myuser
DB_HOST=localhost
DB_PORT=5432
RABBITMQ_USER=myuser
RABBITMQ_PASSWORD=mypassword
```

Replace the placeholder values with your actual configuration values.

### 2. Create the `env_vars.yml` File

Create a file named `env_vars.yml` in the same directory as your `main.yaml` playbook with the following content:

```yaml
---
repo_url: "{{ lookup('env', 'REPO_URL') }}"
branch: "{{ lookup('env', 'BRANCH') }}"
app_dir: "{{ lookup('env', 'APP_DIR') }}"
pg_password: "{{ lookup('env', 'PG_PASSWORD') }}"
db_name: "{{ lookup('env', 'DB_NAME') }}"
db_user: "{{ lookup('env', 'DB_USER') }}"
db_host: "{{ lookup('env', 'DB_HOST') }}"
db_port: "{{ lookup('env', 'DB_PORT') }}"
rabbitmq_user: "{{ lookup('env', 'RABBITMQ_USER') }}"
rabbitmq_password: "{{ lookup('env', 'RABBITMQ_PASSWORD') }}"
```

### 3. Create the `main.yaml` Playbook

Save the following Ansible playbook content in a file named `main.yaml`:

```yaml
---
- name: Automated Deployment and Configuration with Ansible for NESTJS
  hosts: hng
  become: yes
  vars_files:
    - ./env_vars.yml  # Include the environment variables file

  tasks:
    - name: Update apt package index
      apt:
        update_cache: yes
    
    - name: Install required packages
      apt:
        name:
          - git
          - python3-pip
          - nginx
          - postgresql
          - postgresql-contrib
          - rabbitmq-server
          - nodejs
          - npm
        state: present

    - name: Add hng user with sudo privileges
      user:
        name: hng
        shell: /bin/bash
        groups: sudo
        append: yes

    - name: Clone the devops branch of the repository
      git:
        repo: "{{ repo_url }}"
        dest: "{{ app_dir }}"
        version: "{{ branch }}"
        force: yes
      become_user: hng

    - name: Ensure /var/secrets directory exists
      file:
        path: /var/secrets
        state: directory
        mode: '0755'

    - name: Save PostgreSQL admin credentials
      copy:
        dest: /var/secrets/pg_pw.txt
        content: "{{ db_user }}:{{ pg_password }}"
        mode: '0600'

    - name: Ensure /var/log/stage_5b directory exists
      file:
        path: /var/log/stage_5b
        state: directory
        mode: '0755'

    - name: Install Node.js dependencies
      npm:
        path: "{{ app_dir }}"
        state: present
        production: yes

    - name: Configure PostgreSQL database
      postgresql_db:
        name: "{{ db_name }}"
        state: present
      postgresql_user:
        name: "{{ db_user }}"
        password: "{{ pg_password }}"
        priv: "ALL"
        state: present

    - name: Configure RabbitMQ
      rabbitmq_user:
        user: "{{ rabbitmq_user }}"
        password: "{{ rabbitmq_password }}"
        vhost: /
        configure_priv: .*
        read_priv: .*
        write_priv: .*
        state: present

    - name: Configure environment variables
      copy:
        dest: "{{ app_dir }}/.env"
        content: |
          DATABASE_URL=postgres://{{ db_user }}:{{ pg_password }}@{{ db_host }}:{{ db_port }}/{{ db_name }}
          RABBITMQ_URL=amqp://{{ rabbitmq_user }}:{{ rabbitmq_password }}@localhost:5672/
          PORT=3000
        mode: '0644'
      become: yes

    - name: Start the application
      command: "nohup npm run start:prod > /var/log/stage_5b/out.log 2> /var/log/stage_5b/error.log &"
      args:
        chdir: "{{ app_dir }}"
      become_user: hng

    - name: Set ownership of log files
      file:
        path: "/var/log/stage_5b"
        state: directory
        recurse: yes
        owner: hng
        group: hng

    - name: Configure Nginx to reverse proxy
      copy:
        dest: /etc/nginx/sites-available/stage_5b
        content: |
          server {
              listen 80;
              server_name _;

              location / {
                  proxy_pass http://127.0.0.1:3000;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
              }
          }
      notify: Restart nginx

    - name: Enable Nginx site
      file:
        src: /etc/nginx/sites-available/stage_5b
        dest: /etc/nginx/sites-enabled/stage_5b
        state: link

    - name: Remove default Nginx site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### 4. Create an Inventory File

Create a file named `inventory.cfg` with the following content (replace `your_server_ip` with the actual IP address of your server and `your_user` with your SSH username):

```ini
[hng]
your_server_ip ansible_user=your_user
```

### 5. Load Environment Variables

Before running the playbook, load the environment variables from your local `.env` file:

```sh
export $(cat .env | xargs)
```

### 6. Run the Playbook

Run the Ansible playbook:

```sh
ansible-playbook -i inventory.cfg main.yaml -b
```

## Summary

This setup automates the deployment and configuration of the NESTJS boilerplate on a remote Ubuntu 22.04 server. It ensures that all sensitive values and configuration parameters are securely managed through environment variables.

### Key Steps

1. **Create a `.env` file** on your local machine with all the necessary variables.
2. **Create the `env_vars.yml` file** to load environment variables into the playbook.
3. **Create the `main.yaml` Ansible playbook** with the necessary tasks.
4. **Create an inventory file** to specify the target server.
5. **Load the environment variables** before running the playbook.
6. **Run the Ansible playbook** to configure the remote server.

This ensures a reproducible and automated deployment process for your NESTJS application.
