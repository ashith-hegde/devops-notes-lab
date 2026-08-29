# Ansible Templating with Jinja2

## The Templating Engine

A **templating engine** is a tool that combines a template with variables and expressions to produce a finished output.

Ansible uses templating extensively to customize tasks and generate configuration files dynamically.

```text
Template + Variables
        |
        v
     Jinja2
        |
        v
Rendered Output
```

For example, the same template can be used across multiple servers while each server receives values appropriate to its environment.

### Common Uses

- Playbooks and task definitions — dynamically use values during a run.
- Configuration files — generate configuration files instead of manually editing each server.
- Per-host configuration — variables can be evaluated for each host at runtime.

---

# What is Jinja2?

**Jinja2** is a Python-based templating engine used by Ansible.

It provides syntax for:

- Variables
- Expressions
- Conditions
- Loops
- Filters
- Data manipulation

Ansible uses Jinja2 as its templating language and extends it with Ansible-specific filters and functionality.

---

# Variables and Expressions

Jinja2 uses **double curly braces** to evaluate a variable or expression:

```jinja2
{{ variable_name }}
```

Example:

```jinja2
The name is {{ my_name }}
```

If:

```yaml
my_name: Bond
```

the rendered result is:

```text
The name is Bond
```

---

# Jinja2 Filters

Filters transform or process a value.

The pipe character (`|`) passes the value to a filter:

```jinja2
{{ my_name | upper }}
```

For example, if:

```yaml
my_name: Bond
```

then:

| Filter | Expression | Result |
|---|---|---|
| `upper` | `{{ my_name | upper }}` | `BOND` |
| `lower` | `{{ my_name | lower }}` | `bond` |
| `title` | `{{ my_name | title }}` | `Bond` |
| `replace` | `{{ my_name | replace("Bond", "Bourne") }}` | `Bourne` |

Filters can also be chained:

```jinja2
{{ my_name | lower | title }}
```

---

# Providing a Safe Default

A variable may not always be defined.

The `default` filter provides a fallback:

```jinja2
{{ first_name | default("James") }}
```

If `first_name` is undefined:

```text
James
```

If:

```yaml
first_name: John
```

the result is:

```text
John
```

Conceptually:

```text
Variable defined?
       |
   +---+---+
   |       |
  Yes      No
   |       |
   v       v
 use it   default
```

---

# Filters for Lists and Sets

Jinja2 and Ansible provide filters that can work with lists and sets.

Example:

```yaml
numbers:
  - 1
  - 2
  - 3
  - 2
```

### `min`

```jinja2
{{ numbers | min }}
```

Result:

```text
1
```

### `max`

```jinja2
{{ numbers | max }}
```

Result:

```text
3
```

### `unique`

```jinja2
{{ numbers | unique | list }}
```

Result:

```text
[1, 2, 3]
```

`unique` removes duplicate values. Using `| list` converts the result into a normal list.

---

# Combining Lists with `union`

The `union` filter combines two lists and returns unique values.

```jinja2
{{ [1, 2, 3, 4] | union([4, 5]) }}
```

Result:

```text
[1, 2, 3, 4, 5]
```

---

# Finding Common Values with `intersect`

The `intersect` filter returns values shared by two lists.

```jinja2
{{ [1, 2, 3, 4] | intersect([4, 5]) }}
```

Result:

```text
[4]
```

> The correct filter name is `intersect`, not `interest`.

---

# Selecting a Random Value

The `random` filter can select a random item from a sequence.

```jinja2
{{ range(1, 200) | random }}
```

This could produce:

```text
137
```

The result is **one value**, not a list.

---

# Joining Values

The `join` filter combines elements of a list into a single string.

```jinja2
{{ ['a', 'b', 'c'] | join('') }}
```

Result:

```text
abc
```

Using a separator:

```jinja2
{{ ['a', 'b', 'c'] | join(', ') }}
```

Result:

```text
a, b, c
```

---

# Loops in Jinja2 Templates

Jinja2 supports loops using block syntax.

```jinja2
{% for number in [1, 2, 3] %}
{{ number }}
{% endfor %}
```

Rendered output:

```text
1
2
3
```

The important syntax is:

```jinja2
{% ... %}
```

This represents a **Jinja2 statement/block**, rather than a value being directly rendered.

Compare:

```jinja2
{{ variable }}
```

with:

```jinja2
{% for item in items %}
...
{% endfor %}
```

A useful rule:

- `{{ ... }}` → evaluate and output a value
- `{% ... %}` → perform template logic

---

# Conditions in Jinja2 Templates

Jinja2 also supports conditional logic.

```jinja2
{% for number in [1, 2, 3] %}
{% if number == 2 %}
{{ number }}
{% endif %}
{% endfor %}
```

Rendered output:

```text
2
```

The structure is:

```jinja2
{% if condition %}
    ...
{% endif %}
```

Conditions can also be combined with loops when generating configuration files.

---

# Jinja2 Syntax at a Glance

| Syntax | Purpose | Example |
|---|---|---|
| `{{ ... }}` | Output a value/expression | `{{ hostname }}` |
| `{% ... %}` | Template logic/block | `{% if enabled %}` |
| `{# ... #}` | Comment | `{# This is a comment #}` |

---

# Ansible Extends Jinja2

Jinja2 is designed to be extensible, and Ansible provides additional filters and functionality useful for infrastructure automation.

Examples include filters for:

- YAML and JSON conversion
- File path manipulation
- Regular expressions
- Password hashing
- Data transformation

For example:

```jinja2
{{ data | to_json }}
```

converts data to JSON.

Another example:

```jinja2
{{ path | basename }}
```

extracts the final component of a file path.

These Ansible-specific filters make Jinja2 particularly useful for configuration management and infrastructure automation.

---

# How Jinja2 Works in Ansible

A simplified view is:

```text
Playbook / Template
        +
Variables and facts
        |
        v
   Jinja2 rendering
        |
        v
   Rendered values
        |
        v
 Ansible executes the task
```

For a template file, Ansible can combine:

```text
Template
+
Host-specific variables
+
Ansible facts
+
Other available data
```

to produce a host-specific configuration file.

For example:

```jinja2
server_name {{ inventory_hostname }}
```

If the current host is:

```text
web01
```

the rendered content becomes:

```text
server_name web01
```

> Ansible does **not** literally create a completely new copy of the entire playbook with every variable substituted before execution. Templating occurs as Ansible processes the relevant task, argument, or template content.

---

# Templates and Configuration Management

One of the most important practical uses of Jinja2 in Ansible is generating configuration files.

Example template:

```jinja2
server {
    listen {{ http_port }};
    server_name {{ server_name }};
}
```

Variables:

```yaml
http_port: 8080
server_name: example.com
```

Rendered configuration:

```text
server {
    listen 8080;
    server_name example.com;
}
```

The template defines **how the configuration should look**, while variables provide **the values that change between environments or hosts**.

---

# Core Mental Model

```text
Template
   +
Variables
   +
Jinja2 logic / filters
   |
   v
Rendered output
```

> **The template defines the format, variables provide the values, and Jinja2 brings them together.**

---

# Official Documentation

For the complete and current list of Jinja2 and Ansible filters, syntax, templating behavior, and examples, refer to the official Ansible documentation:

https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_filters.html

The official documentation is especially useful because Ansible provides additional filters and functionality on top of standard Jinja2.

When working on real playbooks, check the documentation for the version of Ansible you are actually using.

