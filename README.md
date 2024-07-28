# Automated Deployment and Configuration with Ansible for NESTJS Boilerplate

This repository contains an Ansible playbook that automates the deployment and configuration of the NESTJS boilerplate on a remote Ubuntu 22.04 server.

## Prerequisites

1. **Ansible**: Install Ansible on your local machine.

   ```sh
   sudo apt update
   sudo apt install ansible
   ```

2. **SSH Access**: Ensure SSH access to your remote server with sudo privileges.

### 1. Create the `main.yaml` Playbook

Save the following Ansible playbook content in a file named `main.yaml`:

```yaml
---
---
- name: Automated Deployment and Configuration with Ansible for NESTJS
  hosts: hng
  become: yes
  vars:
    repo_url: 'https://github.com/hngprojects/hng_boilerplate_nestjs.git'
    branch: 'devops'
    app_dir: '/opt/stage_5b'
    pg_password: 'database_password'  # replace with a secure password or generate dynamically
    db_name: 'database_name'
    db_user: 'database_user'
    db_host: 'localhost'
    db_port: 5432

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
        user: myuser
        password: mypassword
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
          RABBITMQ_URL=amqp://myuser:mypassword@localhost:5672/
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

### 2. Create an Inventory File

Create a file named `inventory.cfg` with the following content (replace `your_server_ip` with the actual IP address of your server and `your_user` with your SSH username):

```ini
[hng]
your_server_ip ansible_user=your_user
```

### 3. Run the Playbook

Run the Ansible playbook:

```sh
ansible-playbook -i inventory.cfg main.yaml -b
```

## Summary

This setup automates the deployment and configuration of the NESTJS boilerplate on a remote Ubuntu 22.04 server.

### Key Steps

1. **Create the `main.yaml` Ansible playbook** with the necessary tasks.
2. **Create an inventory file** to specify the target server.
3. **Run the Ansible playbook** to configure the remote server.

This ensures a reproducible and automated deployment process for your NESTJS application.
