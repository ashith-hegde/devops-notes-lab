# Ansible Roles

## What Are Ansible Roles?

Ansible roles help keep playbooks organized, reusable, and easier to maintain.

A role packages related automation for a particular purpose into a reusable structure. Instead of placing all tasks, variables, templates, and handlers inside one large playbook, they can be organized inside a role and reused by name.

```text
Playbook
   |
   +-- mysql role
   +-- nginx role
   +-- monitoring role
```

Roles are useful for packaging the automation required to configure a:

- Database server
- Web server
- Redis server
- Backup server

> A role describes what configuration or responsibility should be applied. A server can use one or more roles depending on its requirements.

---

# Example: Using a Role

```yaml
---
- name: Install and configure MySQL
  hosts: db_servers

  roles:
    - mysql
```

The playbook references the role by name. The actual tasks and supporting files live inside the `mysql` role.

This keeps the playbook simple while allowing the implementation to remain organized and reusable.

---

# Why Use Roles?

Roles are not only a way to avoid repetition. They also encourage a consistent project structure.

Instead of placing everything in one playbook, a role separates different types of content into dedicated directories.

Benefits include:

- Better organization
- Reusability
- Easier maintenance
- Easier sharing
- Clearer separation of responsibilities

---

# Common Role Structure

A typical role can contain directories such as:

```text
mysql/
├── tasks/
├── handlers/
├── templates/
├── vars/
└── defaults/
```

## `tasks/`

Contains the steps performed by the role.

The main task file is commonly:

```text
tasks/main.yml
```

Example:

```yaml
- name: Install MySQL
  ansible.builtin.package:
    name: mysql-server
    state: present
```

## `vars/`

Contains variables used by the role.

Example:

```yaml
mysql_package: mysql-server
```

Variables in `vars/` generally have higher precedence than variables in `defaults/`.

## `defaults/`

Contains default values that users of the role can more easily override.

Example:

```yaml
mysql_port: 3306
```

## `templates/`

Contains Jinja2 templates that can be filled with variables at runtime.

Example:

```text
templates/my.cnf.j2
```

## `handlers/`

Contains handlers used by the role.

For example:

```yaml
- name: Restart MySQL
  ansible.builtin.service:
    name: mysql
    state: restarted
```

---

# Example Role Directory

```text
roles/
└── mysql/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── my.cnf.j2
    ├── vars/
    │   └── main.yml
    └── defaults/
        └── main.yml
```

Each directory has a specific purpose, making the automation easier to understand.

---

# Ansible Galaxy

Ansible Galaxy is a hub for discovering and sharing Ansible content.

It provides community and vendor-provided content, including roles and collections.

Available content can cover:

- Web servers
- Database servers
- Monitoring
- Security software
- Infrastructure automation
- Other applications and platforms

Before building a role from scratch, check whether existing content already provides the required functionality.

However, review third-party content before using it, especially in production environments.

---

# Creating a Role Skeleton

The `ansible-galaxy` command can generate the basic structure for a new role.

```bash
ansible-galaxy init <role_name>
```

Example:

```bash
ansible-galaxy init mysql
```

This creates a role skeleton containing standard directories and files.

---

# Where Does Ansible Look for Roles?

Ansible searches configured role paths when resolving role names.

A common project-local location is:

```text
roles/
```

Another possible location is:

```text
/etc/ansible/roles
```

Additional locations can be configured through Ansible's `roles_path` setting.

The exact search path depends on the active Ansible configuration.

To inspect role-related settings:

```bash
ansible-config dump | grep ROLE
```

A more targeted check can be:

```bash
ansible-config dump | grep DEFAULT_ROLES_PATH
```

---

# Installing a Role from Ansible Galaxy

A role can be installed using:

```bash
ansible-galaxy role install <role_name>
```

Example:

```bash
ansible-galaxy role install geerlingguy.mysql
```

Ansible downloads and installs the role into the configured roles path unless another path is specified.

> `ansible-galaxy install <name>` is older shorthand usage. `ansible-galaxy role install <name>` is clearer because it explicitly identifies the content as a role.

---

# Installing a Role to a Specific Path

Use `-p` to specify an installation path:

```bash
ansible-galaxy role install geerlingguy.mysql -p ./roles
```

This is useful when maintaining roles inside a project-local directory.

---

# Listing Installed Roles

To list installed roles:

```bash
ansible-galaxy role list
```

The older shorthand may also be available:

```bash
ansible-galaxy list
```

---

# Using More Than One Role

A play can use multiple roles.

```yaml
---
- name: Configure database and web server
  hosts: db_and_webservers

  roles:
    - mysql
    - nginx
```

Roles are applied in the order they are listed.

---

# Passing Options to a Role

A role can also be passed using dictionary syntax.

```yaml
---
- name: Configure MySQL
  hosts: database_servers

  roles:
    - role: mysql
      become: true
      vars:
        mysql_port: 3306
```

This allows role-specific options and variables to be associated with the role.

---

# Important Commands

## Create a Role

```bash
ansible-galaxy init <role_name>
```

## Install a Role

```bash
ansible-galaxy role install <role_name>
```

## Install a Role to a Specific Path

```bash
ansible-galaxy role install <role_name> -p <path>
```

## List Installed Roles

```bash
ansible-galaxy role list
```

## Check Role Configuration

```bash
ansible-config dump | grep ROLE
```

---

# Roles and Playbooks

```text
Playbook
   |
   +-- Defines target hosts
   |
   +-- Defines high-level orchestration
   |
   +-- Includes one or more roles
            |
            +-- Tasks
            +-- Variables
            +-- Defaults
            +-- Templates
            +-- Handlers
```

The playbook describes the broader automation, while roles package reusable parts of that automation.

---

# Key Takeaways

- Roles keep Ansible automation organized and reusable.
- A role packages related tasks, variables, templates, handlers, and supporting content.
- Common role directories include `tasks/`, `vars/`, `defaults/`, `templates/`, and `handlers/`.
- Roles can be referenced by name from a play.
- A single play can use multiple roles.
- Role-specific options and variables can be passed using dictionary syntax.
- `ansible-galaxy init` creates a role skeleton.
- Ansible searches configured role paths to find roles.
- `ansible-config dump | grep ROLE` can help inspect role-related configuration.
- Roles can be discovered and shared through Ansible Galaxy.
- Check existing roles or collections before building from scratch, but review third-party content before production use.
- Package reusable automation once as a role and reference it wherever it is needed.

