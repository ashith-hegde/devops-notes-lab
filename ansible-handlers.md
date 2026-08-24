# Ansible Handlers

## Regular Tasks vs Handlers

### Regular Task

A regular task runs whenever Ansible reaches it during a playbook run. It can therefore run on every playbook execution, subject to conditions and other task controls.

### Handler

A handler is a special task that normally waits until another task notifies it.

A handler is useful when an action should happen **only after a relevant change**. For example, after updating an application configuration, the application may need to restart. If nothing changed, restarting the application is unnecessary.

```text
Task runs
   |
   +-- No change --> Handler is not notified
   |
   +-- Changed ----> Handler is notified
                         |
                         v
                    Handler runs
```

This ties an action, such as restarting a service, to the task that actually changed the system state.

---

# The `notify` Directive

The `notify` directive tells Ansible which handler to notify when a task reports a change.

```yaml
- name: Copy application code
  ansible.builtin.copy:
    src: app_code
    dest: /opt/application/
  notify: Restart Application Service
```

If the `copy` task changes something, Ansible notifies the handler named `Restart Application Service`.

If the task completes without making a change, the handler is normally not notified.

---

# Defining a Handler

Handlers are defined in the `handlers` section of a play.

```yaml
handlers:
  - name: Restart Application Service
    ansible.builtin.service:
      name: application_service
      state: restarted
```

The handler name is referenced by `notify`.

```text
Task
 |
 | notify
 v
Handler Name
 |
 v
Matching Handler
 |
 v
Action is performed
```

The value in `notify` must correctly reference the intended handler.

---

# Complete Example

```yaml
---
- name: Deploy Application
  hosts: application_servers

  tasks:
    - name: Copy application code
      ansible.builtin.copy:
        src: app_code
        dest: /opt/application/
      notify: Restart Application Service

  handlers:
    - name: Restart Application Service
      ansible.builtin.service:
        name: application_service
        state: restarted
```

## What Happens During Execution?

1. Ansible runs the `Copy application code` task.
2. If the task makes a change, it notifies `Restart Application Service`.
3. The handler is queued.
4. Ansible runs the handler at the appropriate handler execution point, normally after the regular tasks in the play.
5. If no change occurs, the handler is not normally triggered.

---

# Why Use Handlers?

Handlers are useful for actions that should happen only when something changes.

Common examples include:

- Restarting a service after its configuration changes
- Reloading a web server after updating its configuration
- Restarting an application after deploying new code
- Reloading a service after updating a certificate or another relevant file

Example:

```yaml
- name: Update web server configuration
  ansible.builtin.copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
  notify: Reload Nginx

handlers:
  - name: Reload Nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

If `nginx.conf` is unchanged, Nginx does not need to be reloaded.

---

# Regular Task vs Handler

| Regular Task | Handler |
|---|---|
| Runs when Ansible reaches the task | Normally waits until notified |
| Can run on every playbook execution | Runs when a notifying task reports a change |
| Used for normal playbook operations | Used for actions triggered by state changes |
| Does not require `notify` | Is typically triggered through `notify` |

---

# Key Takeaways

- A regular task executes when Ansible reaches it.
- A handler is a special task that waits to be notified.
- `notify` connects a task to a handler.
- A task normally notifies a handler only when it reports a change.
- Handlers are useful for restarting or reloading services after relevant changes.
- Handlers are defined under the `handlers` section.
- The `notify` value must correctly reference the intended handler.
- This pattern helps avoid unnecessary operations and fits Ansible's idempotent approach.

## Quick Flow

```text
Configuration task changes a file
            |
            v
       notify handler
            |
            v
     Handler restarts service
```

