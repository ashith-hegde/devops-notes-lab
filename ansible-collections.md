# Ansible Collections

## What Are Ansible Collections?

Ansible Collections are a distribution format for Ansible content.

A collection can package related:

- Modules
- Plugins
- Roles
- Playbooks
- Supporting content

Collections make it easier for vendors, cloud providers, and the Ansible community to distribute specialized automation.

> Think of a collection as a **package of related Ansible automation content**.

---

## Why Collections Exist

`ansible-core` provides the Ansible engine and built-in content. Real environments often need specialized functionality for cloud platforms, network vendors, databases, monitoring, security, and other platforms.

Collections provide a standardized way to distribute this specialized content.

A simplified view:

```text
ansible-core
    |
    +-- Ansible engine
    +-- Built-in content
    |
    +-- Additional collections
            |
            +-- Cloud
            +-- Network
            +-- Database
            +-- Vendor-specific
            +-- Community content
```

---

## Collection Naming

Collections use the following naming convention:

```text
namespace.collection
```

Examples:

```text
community.general
community.docker
ansible.posix
```

The first part is the **namespace** and the second part is the **collection name**.

For vendor collections, the namespace is commonly associated with the vendor or organization.

> Do not confuse this with older role naming conventions. Collections use `namespace.collection`.

---

## Fully Qualified Collection Name (FQCN)

When referring to content from a collection, Ansible uses the **Fully Qualified Collection Name (FQCN)**.

The general format is:

```text
namespace.collection.plugin_name
```

For modules:

```text
community.docker.docker_container
ansible.builtin.copy
```

Example:

```yaml
- name: Create a Docker container
  community.docker.docker_container:
    name: web
    image: nginx
    state: started
```

FQCNs make the source of the module or plugin explicit and reduce ambiguity between similarly named content.

---

## Installing a Collection

Use `ansible-galaxy` to install a collection:

```bash
ansible-galaxy collection install <namespace>.<collection>
```

Example:

```bash
ansible-galaxy collection install community.docker
```

A specific version can also be installed:

```bash
ansible-galaxy collection install community.docker:4.0.0
```

For production projects, collection upgrades should be tested rather than blindly installing the latest version.

---

## Installing Collections from `requirements.yml`

For a reusable project, external collection dependencies can be declared in a `requirements.yml` file.

Example:

```yaml
---
collections:
  - name: community.docker
  - name: community.general
  - name: ansible.posix
```

Install them with:

```bash
ansible-galaxy collection install -r requirements.yml
```

This makes project dependencies easier to document and reproduce.

Versions can also be specified:

```yaml
---
collections:
  - name: community.docker
    version: ">=4.0.0"

  - name: community.general
    version: ">=10.0.0"
```

The exact version constraints should be selected based on the project's compatibility requirements.

---

## Roles and Collections in the Same Requirements File

A `requirements.yml` file can contain both roles and collections:

```yaml
---
roles:
  - name: geerlingguy.nginx

collections:
  - name: community.general
  - name: community.docker
```

The appropriate `ansible-galaxy` commands can then be used to install the declared dependencies.

---

## `ansible.builtin`

`ansible.builtin` is the namespace for Ansible's built-in content.

Examples:

```yaml
ansible.builtin.copy
ansible.builtin.service
ansible.builtin.command
ansible.builtin.debug
```

A good practice is to use FQCNs consistently:

```yaml
- name: Restart service
  ansible.builtin.service:
    name: nginx
    state: restarted
```

rather than:

```yaml
- name: Restart service
  service:
    name: nginx
    state: restarted
```

The short form may work, but the FQCN clearly identifies where the module comes from.

---

## Collection Structure

A simplified collection can look like:

```text
my_namespace/
└── my_collection/
    ├── galaxy.yml
    ├── README.md
    ├── plugins/
    │   ├── modules/
    │   ├── inventory/
    │   └── ...
    ├── roles/
    ├── playbooks/
    ├── docs/
    ├── tests/
    └── meta/
```

The `galaxy.yml` file contains collection metadata such as:

- Namespace
- Collection name
- Version
- Other collection metadata

---

## Collection Dependencies

Collections can depend on other collections.

```text
Collection A
    |
    +---- depends on ----> Collection B
```

Dependencies can be declared in collection metadata.

This is another reason to manage collection dependencies and versions explicitly.

---

## Where Collections Are Installed

A common user-level collection location is:

```text
~/.ansible/collections/
```

The exact search paths depend on Ansible configuration.

Useful commands when investigating collection paths include:

```bash
ansible-config dump
```

and:

```bash
ansible-galaxy collection list
```

---

## Listing Installed Collections

To list installed collections:

```bash
ansible-galaxy collection list
```

This is useful when troubleshooting module or plugin availability.

To inspect a specific module:

```bash
ansible-doc community.docker.docker_container
```

`ansible-doc` displays the documentation and examples available for the installed module.

---

## Common Collection Troubleshooting

A common error is:

```text
couldn't resolve module/action
```

For example:

```text
couldn't resolve module/action 'community.docker.docker_container'
```

Possible causes include:

1. The collection is not installed.
2. The FQCN is incorrect.
3. The installed collection version does not provide the expected module.
4. Ansible is searching a different collection path.
5. A collection dependency is missing.

Useful checks:

```bash
ansible-galaxy collection list
```

```bash
ansible-doc community.docker.docker_container
```

```bash
ansible-config dump
```

---

## Collections in a Project

A reusable project can keep its external collection dependencies in the repository:

```text
ansible-server-usage-monitor/
├── ansible/
├── docker/
├── inventory/
├── requirements.yml
├── README.md
└── .gitignore
```

Example `requirements.yml`:

```yaml
---
collections:
  - name: community.docker
```

Install the dependency with:

```bash
ansible-galaxy collection install -r requirements.yml
```

This is preferable to relying on undocumented manual installation steps.

---

## Collections vs Roles

| Role | Collection |
|---|---|
| Reusable automation structure | Distribution package for Ansible content |
| Organizes tasks, handlers, templates, variables, etc. | Can contain roles, modules, plugins, and playbooks |
| Usually focused on a particular automation responsibility | Can package a broader set of related functionality |

A useful mental model:

```text
Collection
├── Modules
├── Plugins
├── Roles
└── Playbooks
```

A collection can therefore contain roles.

---

## Collections vs `ansible-core`

A simplified view of the relationship:

```text
Ansible
│
├── ansible-core
│   ├── Ansible engine
│   └── ansible.builtin content
│
└── Additional collections
    ├── community.docker
    ├── community.general
    ├── ansible.posix
    └── Vendor collections
```

`ansible-core` provides the Ansible engine and core functionality, while collections provide additional content and integrations.

---

## Common Commands

### Install a Collection

```bash
ansible-galaxy collection install namespace.collection
```

### Install a Specific Version

```bash
ansible-galaxy collection install namespace.collection:1.2.3
```

### Install from Requirements

```bash
ansible-galaxy collection install -r requirements.yml
```

### List Installed Collections

```bash
ansible-galaxy collection list
```

### View Module Documentation

```bash
ansible-doc namespace.collection.module
```

---

## Key Takeaways

- Collections are a distribution format for Ansible content.
- They can contain modules, plugins, roles, playbooks, and supporting content.
- Collections commonly provide vendor, cloud, network, database, and community-specific functionality.
- Collections use the `namespace.collection` naming convention.
- Modules and plugins can be referenced using their FQCN.
- `ansible.builtin` contains Ansible's built-in content.
- Collections can be installed using `ansible-galaxy`.
- Project dependencies can be declared in `requirements.yml`.
- `ansible-galaxy collection list` can be used to inspect installed collections.
- `ansible-doc` can be used to inspect modules and examples.
- Collection versions should be tested and managed carefully in production environments.
- Collections can contain roles, modules, plugins, and playbooks.

---

## Official Documentation

For the complete and current picture, refer to the official Ansible documentation:

- Collections guide: https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_using.html
- Downloading and installing collections: https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_downloading.html
- Collection structure: https://docs.ansible.com/projects/ansible/latest/dev_guide/developing_collections_structure.html
- Ansible Galaxy guide: https://docs.ansible.com/projects/ansible/latest/galaxy/user_guide.html

When working with collections in a real project, prefer the documentation for the Ansible and collection versions you are actually using.

