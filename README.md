# CSEC 473 - Assignment 2: Wordpress Server with Vulnerable MySQL Database 

## Overview

This project utilizes Ansible to deploy a three-tier vulnerable web application MySQL database, Nginx webserver, and WordPress webpage. It prepares the environment by installing necessary packages, configuring network communication between the reverse proxy and application layers, and deploys a custom vulnerable functions.php file containing a deliberate SQL injection directly to the WordPress server. Exploiting the SQL injection will reveal the hidden flag in a secret table within the schema. For competition environments, this playbook offers critical utility by ensuring rapid and consistent challenge deployment. The Ansible playbook is designed with idempotency checks such as verifying if wp-config.php exists before attempting re-installation—allowing organizers to repair or reset specific services without accidentally wiping the entire competition infrastructure. This setup presents a logic error hidden within custom code and forces competitors to analyze the application behavior and enumerate the database structure manually to locate the flag.

## Vulnerability Description

The vulnerability implemented in this deployment is an unauthenticated UNION-based SQL Injection (SQLi) located within a custom REST API endpoint. The exploit utilizes insecure string concatenation to build database queries rather than using WordPress's built-in prepared statements. For example, an attacker could pass a crafted value like ```' UNION SELECT 1,2,3,4,5,6,7,8,9,version()--``` as a parameter which could expose database metadata, user credentials, or other sensitive tables depending on the parameter. This vulnerability is significant because it allows an attacker to manipulate the backend SQL query structure via the URL search parameter. By injecting specific SQL commands, an attacker can bypass the intended query logic to inspect the database schema and retrieve data from arbitrary tables. This project was created with the purpose of being run on machines competing in a A/D CTF. The implemented vulnerability allows the exfiltration of the flag hidden within the custom wp_flags table.

## Prerequisites

- Target OS: Ubuntu 22.04 LTS
- Ansible version: 2.16.3 
- Required control machine packages: ansible, ansible-lint, sshpass, git
- Packages installed by Ansible: php, php-fpm, php-mysql, php-curl, php-gd, php-mbstring, php-xml, php-xmlrpc, php-soap, php-intl, php-zip, WP-CLI, python3-pymysql

## Quick Start Commands

### Step 1: Clone the GitHub Repository to the Target Machine.

In a terminal, run:

```bash
git clone https://github.com/BSparacio/Ansible_Vulnerable_WordPress.git
```

### Step 2: Update Inventory With Correct Server IPs

Open `inventory.ini` and update the IP addresses to match the target server:

Expected file layout:

```ini
[webserver]
<HOSTNAME> ansible_host=<YOUR_TARGET_IP>

[wordpress]
<HOSTNAME> ansible_host=<YOUR_TARGET_IP>

[database]
<HOSTNAME> ansible_host=<YOUR_TARGET_IP>
```

### Step 2b: Update Global Variables With Correct Server IP

Open `group_vars/all.yml` and update the IP addresse assocaited with the target_ip_address variable:

![Alt text](../screenshots/deployment/change_global_ip_variable.png)

### Step 3: Set Up SSH Keys (if you don't have one).

In a terminal, run:

```bash
./setup-ssh.sh
```

### Step 4: Run the playbook.yml File.

In a terminal, run:
```bash
ansible-playbook playbook.yml 
```

## Documentation
→ [Go to DEPLOYMENT.md](docs/DEPLOYMENT.md)

→ [Go to EXPLOITATION.md](docs/EXPLOITATION.md)

## Competition Use Cases

### Red Team Scenarios

This environment is ideal for practicing web application enumeration and manual SQL injection exploitation. It challenges students to understand the underlying logic of UNION-based attacks, column balancing, and database schema enumeration. It also serves as a target for lateral movement if the database server is segmented from the web server or privilege escalation if the wp_users table is exposed and passwords are cracked.

### Blue Team Scenarios

Defenders can use this environment to practice log analysis and incident response. The Nginx access logs will show the injection attempts, allowing Blue Teams to write regex signatures for detection. There is also an opportunity for Blue Team to remediate the vulnerability. Defenders can patch the functions.php file with ```$wpdb->prepare()``` to use Prepared Statements instead of string concatenation. Disabling the information_schema access or tightening user privileges will also prevent this expoit from occurring.

### Gray Team

For competition organizers, this playbook is a means of reliably deploying challenges to competition machines. Because the infrastructure is defined as code, the environment can be reset or redeployed instantly between rounds or if the event of an emergency if the service is broken by Blue or Red team. This approach ensures a fair and consistent playing field for both teams.

## Technical Details

The Ansible playbook uses a modular role-based approach to configure the environment.

### Play 1: MySQL Database

Targets the database layer, installing MySQL and creating a dedicated wordpress database and user. It also creates a custom wp_flags table and inserts the capture flag, ensuring the objective exists outside the standard WordPress schema.

### Play 2: WordPress

Configures the application layer. It installs PHP and the WordPress core files. It utilizes the wp-cli tool to configure the site details and manage users. Most importantly, it programmatically generates the neon theme by writing index.php, style.css, and functions.php directly to the server. This ensures the vulnerable code and the cyberpunk aesthetic are hardcoded into the deployment without requiring external downloads.

### Play 3: NGINX Webserver

Sets up the Nginx web server as a reverse proxy, handling HTTP requests and forwarding PHP processing to the WordPress application. It manages the virtual host configuration to ensure the site resolves correctly on port 80.

## Troubleshooting

### "Permission Denied" Errors
- Make sure you ran `./setup-ssh.sh` first
- Check that your SSH key was copied successfully

### Can't Connect to Server
- Verify your IP addresses in `inventory.ini`
- Make sure you're connected to the CyberRange network
