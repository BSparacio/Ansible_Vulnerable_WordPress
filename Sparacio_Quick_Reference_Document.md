# CSEC 473 - Assignment 2: Reference Document

Author: Brian Sparacio

## What I Built

This project utilizes Ansible to deploy a three-tier vulnerable web application consisting of a MySQL database, Nginx webserver, and a WordPress webpage. The infrastructure is designed to simulate a Capture The Flag (CTF) environment where the objective is to exploit a logical error to retrieve a hidden flag. The architecture separates concerns across three distinct layers:

1. **MySQL Database:** Stores the WordPress content and a custom wp_flags table containing the capture flag.

2. **WordPress Application:** Runs the PHP application logic and includes a custom "Neon" theme with a deliberate vulnerability.

3. **Nginx Webserver:** Acts as a reverse proxy, handling HTTP requests and forwarding PHP processing to the application layer.

The Ansible playbook uses a modular role-based approach to configure the environment. It programmatically generates the "Neon" theme by writing index.php, style.css, and a vulnerable functions.php directly to the server. This ensures the vulnerable code and the cyberpunk aesthetic are hardcoded into the deployment without requiring external downloads. The setup is designed with idempotency checks, such as verifying if wp-config.php exists before attempting re-installation, allowing for rapid resets of the competition infrastructure.

## Deployment Summary

The deployment process is fully automated via Ansible, requiring only a control node with SSH access to the target environment. The deployment process targets an Ubuntu 22.04 LTS environment and requires Ansible 2.16.3.

### 1. Installation and Configuration

First, clone the repository to the control machine and ensure your inventory.ini file maps the correct IP addresses to the required groups.

```bash
git clone https://github.com/BSparacio/Ansible_Vulnerable_WordPress.git
cd Ansible_Vulnerable_WordPress
```

### 2. SSH Key Setup

Because the environment utilizes a jump host (ssh.cyberrange.rit.edu), simple SSH copying is insufficient. I included a script, setup-ssh.sh, which installs sshpass if necessary and copies the SSH identity to the remote servers via the proxy.

```bash
./setup-ssh.sh
```

### 3. Execution and Validation

Once the keys are distributed, execute the main playbook to provision the stack. This will install Nginx, PHP, and MySQL, and configure the networking between the tiers (e.g., Database listening on port 3306, WordPress on 9000).

```bash
ansible-playbook playbook.yml
```

After the playbook completes, you can verify the deployment using the validation playbook, which checks that services are active and the custom files are in place.

```bash
ansible-playbook validate.yml
```

## Exploitation Summary

The vulnerability implemented in this deployment is an unauthenticated UNION-based SQL Injection (SQLi) located within a custom REST API endpoint. The vulnerability exists in the /ctf/v1/search endpoint registered in the active functions.php file.

### The Vulenrability

The application uses insecure string concatenation to build database queries rather than using WordPress's built-in prepared statements. The q parameter from the HTTP GET request is directly concatenated into the SQL command.

### Vulnerable Code Snippet

```bash
$term = $request['q'];
// VULNERABLE QUERY
$query = "SELECT * FROM wp_users WHERE user_login LIKE '%" . $term . "%'";
```

### Exploitation Strategy

Because the input is not sanitized, an attacker can append SQL commands to the URL. A spike in 500 Internal Server Errors is indicative of failed injection attempts, while successful enumeration allows the attacker to view the wp_flags table. To exploit this, an attacker would navigate to the WordPress webpage and use the search box to execute commands. Information_schema is a standardized, read-only view of metadata about the database objects within a system. It acts as a central catalog containing information about tables, columns, views, constraints, stored procedures, and other database components. By injecting a UNION statement, the attacker can combine the results of the original query with a selection from the hidden table. 

### Mitigation

The primary fix for this vulnerability is to patch the functions.php file to use WordPress's built-in prepared statements ($wpdb->prepare). This ensures that the input is treated as a string literal rather than executable SQL command.

### Secure Code Patch:

```bash
$query = $wpdb->prepare(
    "SELECT * FROM wp_users WHERE user_login LIKE %s",
    '%' . $wpdb->esc_like($term) . '%' 
);
```

## File Tree Layout

```
.
├── README.md
├── ansible.cfg
├── docs
│   ├── DEPLOYMENT.md
│   └── EXPLOITATION.md
├── group_vars
│   └── all.yml
├── inventory.ini
├── playbook.yml
├── roles
│   ├── mysql
│   │   ├── handlers
│   │   │   └── main.yml
│   │   └── tasks
│   │       └── main.yml
│   ├── nginx
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── wordpress.conf.j2
│   └── wordpress
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── wp-config.php.j2
├── screenshots
│   └── deployment
│       ├── IP_address_of_target_machine.png
│       ├── ansible_successful_completion.png
│       ├── change_global_ip_variable.png
│       ├── cloning_repo_to_deploy_box.png
│       ├── deployment_verify_mysql.png
│       ├── deployment_verify_nginx.png
│       ├── deployment_verify_php.png
│       ├── enumerate_columns.png
│       ├── enumerate_columns_found_id_and_flag_col.png
│       ├── enumerate_tables.png
│       ├── enumerate_tables_found_flags_table.png
│       ├── enumerate_tables_found_users_table.png
│       ├── exfiltrate_data_found_flag.png
│       ├── inventory_file_edited.png
│       ├── nmap_verify.png
│       ├── playbook_beginning_to_run.png
│       ├── post_exploitation_users_table.png
│       ├── recon_developer_tools_menu.png
│       ├── recon_developer_tools_menu_GET_visible.png
│       ├── recon_developer_tools_menu_SQLI_possible.png
│       ├── running_validate_yml.png
│       ├── successful_ssh_connection.png
│       └── wordpress_deployed_success.png
├── setup-ssh.sh
└── validate.yml
```

## GitHub Repository Link
https://github.com/BSparacio/Ansible_Vulnerable_WordPress.git