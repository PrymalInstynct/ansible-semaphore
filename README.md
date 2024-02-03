# Ansible Semaphore
How to deploy, configure, and use [Ansible Semaphore](https://www.semui.co/)

[Semaphore Official Documentation](https://docs.semui.co/)

[Semaphore GitHub](https://github.com/ansible-semaphore/semaphore)

Ansible Semaphore can be deployed as a Snap, binary package, or Container.

This guide will cover deployment as a Container.

## Prerequisites

- A Host running one of the follow Operating Systems
    - Debian 12+
    - Ubuntu 22.04+
    - Red Hat Enterprise Linux 8+
    - Fedora 38+

- Docker v20+, docker-compose
    - [Docker Install Instructions](https://docs.docker.com/engine/install/)

    **OR**

- Podman v4.10+, podman-docker, docker-compose, podman-compose
    - [Podman Compose Install Instructions](https://docs.oracle.com/en/learn/podman-compose/index.html#introduction)

## Installation
### Create compose.yml
- From the host with docker/podman installed
- `mkdir -p /opt/semaphore`
- `cd /opt/semaphore`
- `vim compose.yml`

    ```yaml
    services:
    postgres:
        restart: unless-stopped
        image: postgres:14
        hostname: postgres
        volumes:
        - semaphore-postgres:/var/lib/postgresql/data
        environment:
        POSTGRES_USER: semaphore
        POSTGRES_PASSWORD: semaphore
        POSTGRES_DB: semaphore
    semaphore:
        restart: unless-stopped
        ports:
        - 3000:3000
        image: semaphoreui/semaphore:latest
        environment:
        SEMAPHORE_DB_USER: semaphore
        SEMAPHORE_DB_PASS: semaphore
        SEMAPHORE_DB_HOST: postgres
        SEMAPHORE_DB_PORT: 5432
        SEMAPHORE_DB_DIALECT: postgres
        SEMAPHORE_DB: semaphore
        SEMAPHORE_PLAYBOOK_PATH: /tmp/semaphore/
        SEMAPHORE_ADMIN_PASSWORD: changeme
        SEMAPHORE_ADMIN_NAME: admin
        SEMAPHORE_ADMIN_EMAIL: admin@localhost
        SEMAPHORE_ADMIN: admin
        SEMAPHORE_ACCESS_KEY_ENCRYPTION: f1kDpdPTVupIpxt1cjmgF8L8i/NSSerPqjrET1IyhHQ=
        SEMAPHORE_LDAP_ACTIVATED: no # if you wish to use ldap, set to: 'yes' 
        SEMAPHORE_LDAP_HOST: dc01.local.example.com
        SEMAPHORE_LDAP_PORT: "636"
        SEMAPHORE_LDAP_NEEDTLS: yes
        SEMAPHORE_LDAP_DN_BIND: uid=bind_user,cn=users,cn=accounts,dc=local,dc=shiftsystems,dc=net
        SEMAPHORE_LDAP_PASSWORD: ldap_bind_account_password
        SEMAPHORE_LDAP_DN_SEARCH: dc=local,dc=example,dc=com
        SEMAPHORE_LDAP_SEARCH_FILTER: (&(uid=%s)(memberOf=cn=ipausers,cn=groups,cn=accounts,dc=local,dc=example,dc=com))
        depends_on:
        - postgres
    volumes:
    semaphore-postgres: null
    ```

- Save and Quit
- `head -c32 /dev/urandom | base64`
- Copy output value to the copy buffer
- `vim compose.yml`
- Replace the variable value `SEMAPHORE_ACCESS_KEY_ENCRYPTION` with the value in you copy buffer
- If utilizing LDAP authentication change the variable value of `SEMAPHORE_LDAP_ACTIVATED` to true
- Update all of the `SEMAPHORE_LDAP_*` to match your environment
    - Take into account how to safely store all Secrets like `SEMAPHORE_LDAP_PASSWORD`, `SEMAPHORE_ADMIN_PASSWORD`, `SEMAPHORE_DB_PASS`
- Save and Quit

### Deploy Ansible Semaphore
- `docker compose up -d`
- `docker ps -a`
- Validate `semaphore-semaphore-1` and `semaphore-postgres-1` are in an Up/Running state

## Configuration
- From a workstation with a web browser
- Navigate to http://IP_Address:3000/auth/login
    - Username: admin
    - Password: changeme
- Select `Create Project`
    - Project Name: Lab STIGs
- Click `Create`

### Change the admin password
- `Select admin -> Edit Account` in the bottom left corner of the webpage
- Set a new password

### Key Stores

Key Stores enable saving of credentials to be reused across `Task Templates`

**Note:** Keys can be tied to a specific user or the user field can be left blank and the username can specified via inventory variable

#### SSH Key

Used for Host authentication, Git repository authentication, etc.

- Select `Key Store`
- Select `New Key`
    - Key Name: ssh_user
    - Type: SSH Key
    - Username: Leave Blank (Unless you want to tie this Key to a specific username)
    - Passphrase: Leave Blank (Unless the Private Key includes a Passphare)
    - Private Key: Paste in Private SSH Key of user who will be authenticating to remote systems

#### Password

Use for user authentication, privilege escalated via sudo, git repository authentication, ansible vault, etc.

- Select `Key Store`
- Select `New Key`
    - Key Name: sudo_user
    - Type: Login with Password
    - Username: Leave Blank (Unless you want to tie this password to a specific username)
    - Password: Paste in the password used to elevate by the user who will be authenticating to remote systems

- Select `Key Store`
- Select `New Key`
    - Key Name: vault_pass
    - Type: Login with Password
    - Username: Leave Blank
    - Password: Paste in the password used to decrypt an ansible vault


#### None

Use to bypass need for providing Access keys to pull from public https Git repositories

- Select `Key Store`
- Select `New Key`
    - Key Name: https_user
    - Type: None

### Repositories

This will establish the locations where Semaphore will pull Ansible projects into the application for use by `Task Templates`

- Select `Repositories`
- Select `New Repository`
    - Name: RHEL 8 STIG
    - URL: git@gitlab.test.lab:cte/rhel-8-stig.git
        - This value can be a path to a local directory, git over ssh , or git over https
    - Branch: main
    - Access Key: Optional, Select from list of credentials created in previous steps
- Click `Create`

### Environments

Environments enable variables needed to successfully execute an Ansible Playbook to be stored and resused across `Task Templates`

All Environment variables stored must be written in JSON format

```json
{
  "var_available_in_playbook_1": 1245,
  "var_available_in_playbook_2": "test"
}
```

At minimum a `blank` Environment must be stored in Semaphore to be reused across `Task Templates`

- Select `Environment`
- Select `New Environment`
    - Environment Name: all
    - Extra variables: {}
    - Environment variables: Leave Blank
- Click `Save`

### Inventories

The Inventory is used to store a groups of hosts and group/host specific variables that can be resused across `Task Templates`

Three types of Inventories are supported
    - Static: Ansible Inventory using the INI format
    - Static YAML: Ansible Inventory using the YAML format
    - File: File Path to inventory found within a pulled Repository

**NOTE:** Only one set of User and Sudo Credentials can be applied to an inventory, so the ssh keys and passwords must be identical across all hosts. The acception is Usernames can be different as long as the same credentials apply to all users specified in the inventory. (see examples below)

#### Static
- Select `Inventory`
- Select `New Inventory`
    - Name: Dev Lab
    - User Credentials: dev_ssh_user
    - Sudo Credentials: dev_sudo_user
    - Type: Static
        ```
        [website]
        website1.dev.lab ansible_hosts=172.18.8.40
        website2.dev.lab ansible_hosts=172.18.8.41

        [websites:vars]
        ansible_user=webadmin

        [database]
        database1.dev.lab ansible_hosts=172.18.8.42
        database1.dev.lab ansible_hosts=172.18.8.43

        [database:vars]
        ansible_user=dbadmin
        ```
- Click `Create`

#### Static YAML
- Select `Inventory`
- Select `New Inventory`
    - Name: Test Lab
    - User Credentials: test_ssh_user
    - Sudo Credentials: test_sudo_user
    - Type: Static YAML
        ```yaml
        ---
        infrastructure:
        children:
            website:
                vars:
                    ansible_user: webadmin
                hosts:
                    website1.test.lab:
                    ansible_hosts: 172.18.9.40
                    website2.test.lab:
                    ansible_hosts: 172.18.9.41
            database:
                vars:
                    ansible_user: dbadmin
                hosts:
                    database1.test.lab:
                    ansible_hosts: 172.18.9.42
                    database1.test.lab:
                    ansible_hosts: 172.18.9.43
        ```
- Click `Create`

#### File
- Select `Inventory`
- Select `New Inventory`
    - Name: Integration lab
    - User Credentials: int_ssh_user
    - Sudo Credentials: int_sudo_user
    - Type: File
    - Path to inventory file: inventory/inventory.yml
- Click `Create`

### Task Templates

`Task Templates` are used to create repeatable way to execute Ansible Playbooks. They support Ad-Hoc execution as well as Scheduled execution via Cron

Three types of `Task Templates` are supported
    - Task: For executing any type of ansible playbook
    - Build: For executing ansible playbook that build software (Supports defining versioning of software artifacts)
    - Deploy: For executing ansible playbooks that deploy software artifacts (Each Deploy task should be associated with a Build task)

#### Create a View

`Task Templates` can be organized by creating `Views`

- Select `Task Templates`
- Select the `Pencil (Edit)` icon next to `ALL`
- Select `Add View`
    - Name: Dev Lab
- Click the Check Mark
- Repeat as Needed

#### Create a Task Template
- Select `Task Templates`
- Select `New Template`
    - Name: Dev Lab - RHEL 8 STIG
    - Description: Applies the Red Hat Entperise Linux 8 STIG to hosts in the inventory
    - Playbook Filename: playbook/apply-rhel8-stig.yml
    - Inventory: Dev Lab
    - Repository: RHEL 8 STIG
    - Environment: all
    - Vault Password: vault_pass
    - Survey Variables: Leave Blank
    - View: Leave Blank (Select a `View` from the dropdown, if you set that up in the previous step)
    - Cron: Leave Blank (Accepts Cron Time format `45 11 * * *`)
    - Allow CLI Args in Task: Check the box
- Click `Create`

## Task Execution

After completing all of the configuration steps Tasks can be executed Ad-Hoc by following the steps below

- Select `Task Templates`
- Select the `View` which contains the `Task Template` you want run
- Click `Run` under the `Action` column of the `Task Template` you want to run
    - Message: Leave Blank (Can contain a Note about why the Task Template was run Ad-Hoc)
    - Debug: Leave Unchecked (Unless you want the Task output to be Verbose)
    - Diff: Leave Unchecked (Unless you want the Task to execute with the `Diff` Ansible option)
    - Dry Run: Leave Unchecked (Unless you want the Task to execute with the `Dry Run` Ansible option)
    - Advanced: Leave Blank (Unless you want to include additional CLI arguments when the Task executes)
        - Example CLI Argument for Limiting the execution to a specific host from the inventory with tags (Must be a JSON formatted array)
        ```json
        [ "-l database1.dev.lab", "--tag RHEL-08-010010" ]
        ```
- Click `Run`
- At this point a new view within Semaphore will open and show you the live output of the running Tasks

### Reviewing Task History

Semaphore saves the Ansible output from each Task execution and it can be reviewed at any time

#### Dashboard History

The `Dashboard History` view shows all tasks that have run from the Semaphore server by click on the Task number you can review the Ansible output for that task execution

- Select `Dashboard`
- Select the `History` tab
- Click on the `Task ID (#16)` of the task you would like to review

#### Task Template History

A history of Task executions from a specific `Task Template` can by found inside the `Task Template` section of Semaphore

- Select `Task Templates`
- Select the `View` where the `Task Template` is organzied you would like to review the history of
- Select the `Task Template` name
- Click on the `Task ID (#12)` of the task you would like to review
