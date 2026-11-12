# Ansible Playbooks, Roles, YAML & Nginx: Study Guide

A study guide (conspectus) covering Ansible playbooks and roles, YAML syntax, variable precedence, three-tier architecture with Nginx reverse proxying, and practical Ansible examples.

## Table of Contents

1. [Ansible Core Concepts: Playbooks & Roles](#1-ansible-core-concepts-playbooks--roles)
2. [YAML Syntax](#2-yaml-syntax)
3. [Variable Precedence](#3-variable-precedence)
4. [Nginx & Three-Tier Architecture](#4-nginx--three-tier-architecture)
5. [Practical Code Examples](#5-practical-code-examples)
6. [Mock Interview Questions](#6-mock-interview-questions)
7. [Hands-on Practice Assignment](#7-hands-on-practice-assignment)
8. [Useful Links](#8-useful-links)

---

## 1. Ansible Core Concepts: Playbooks & Roles

### Quick Vocabulary

| Term | Definition |
|---|---|
| **Control node** | The machine where Ansible runs. |
| **Managed node** | A target server that Ansible configures (agentless, over SSH/WinRM). |
| **Inventory** | The list of managed nodes and groups (see `inventory.ini`). |
| **Module** | A unit of work (`apt`, `service`, `copy`, `template`). |
| **Task** | One call to a module, with a name and arguments. |
| **Play** | A set of tasks mapped to a group of hosts. |
| **Playbook** | A YAML file containing one or more plays. |
| **Role** | A standardized directory structure that packages tasks, handlers, templates, and variables for reuse. |
| **Handler** | A task that runs only when notified by another task that reported `changed`. |
| **Template** | A Jinja2 file (`.j2`) rendered with variables into a final config file. |

### Playbooks

A **playbook** is a YAML file that describes the desired state of your servers. Where an ad-hoc command runs one module once, a playbook is a saved, repeatable, version-controlled automation.

Anatomy of a playbook:

```yaml
---
- name: Play name               # a play
  hosts: webservers             # which inventory group to target
  become: true                  # escalate privileges (sudo)
  vars:                         # play-level variables
    app_port: 8080

  tasks:                        # ordered list of tasks
    - name: Task name
      ansible.builtin.apt:      # module
        name: nginx             # module arguments
        state: present

  handlers:                     # run only when notified
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Key points:

- Tasks run **in order**, top to bottom, on all targeted hosts.
- Each task should have a descriptive `name`.
- Most modules are **idempotent**: re-running the playbook changes nothing if the system is already in the desired state.
- Run with `ansible-playbook -i inventory.ini playbook.yml`.

### Handlers

A handler is a task that is triggered with `notify`, and it runs **only if the notifying task reported `changed`**. The classic use is reloading a service after its config file changes.

- Handlers run **once, at the end of the play**, even if notified by several tasks.
- Handler names must match the `notify` string exactly.
- This avoids needless restarts. If the config didn't change, Nginx isn't reloaded.

### Templates (Jinja2)

A **template** is a config file with placeholders, rendered per host by the `template` module.

```jinja
server {
    listen {{ nginx_port }};
    server_name {{ inventory_hostname }};
}
```

`{{ nginx_port }}` is replaced by the variable's value when Ansible renders the file. Templates keep one config file adaptable to Dev, Staging, and Production.

### Roles

A **role** is a way to organize a playbook into reusable, self-contained units. Instead of one 300-line playbook, you create roles such as `nginx`, `mysql`, and `app`, and compose them.

Standard role layout (generate with `ansible-galaxy init nginx`):

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml        # the main list of tasks
    ├── handlers/
    │   └── main.yml        # handlers
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/              # static files to copy as-is
    ├── defaults/
    │   └── main.yml        # default variables (lowest precedence, easy to override)
    ├── vars/
    │   └── main.yml        # role variables (high precedence, hard to override)
    └── meta/
        └── main.yml        # metadata and role dependencies
```

Using a role from a playbook:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - nginx
```

### Playbook vs. Role

| Aspect | Playbook | Role |
|---|---|---|
| What it is | A single YAML file with plays | A directory structure |
| Purpose | Entry point: *what runs where* | Reusable unit: *how to configure one component* |
| Contains | Plays, hosts, tasks, roles | Tasks, handlers, templates, defaults, files |
| Reuse | Limited | High (share via Git or Ansible Galaxy) |
| Analogy | The **conductor's score** for the whole concert | A **single instrument's part** that can be used in many pieces |

**Ansible Galaxy** is the public hub for sharing community roles and collections (`ansible-galaxy install <role>`).

---

## 2. YAML Syntax

**YAML** ("YAML Ain't Markup Language") is a human-readable data-serialization format. Ansible playbooks, inventories (YAML format), variable files, and role files are all written in YAML. File extension: `.yml` or `.yaml`.

### Core Rules

1. **Indentation defines structure.** Use **spaces, never tabs**. The convention is **2 spaces** per level.
2. Items at the same level must have **identical indentation**.
3. Keys and values are separated by a colon **followed by a space** (`key: value`).
4. Comments start with `#`.
5. A file typically begins with `---` (document start marker).
6. YAML is **case-sensitive**.

### Key-Value Pairs

```yaml
name: web-server-1
port: 80
enabled: true
```

### Lists (Sequences)

Each item starts with `- ` (dash and space):

```yaml
packages:
  - nginx
  - curl
  - git

# Inline (flow) style
packages: [nginx, curl, git]
```

### Dictionaries (Mappings)

Nested key-value pairs, defined by indentation:

```yaml
server:
  host: 192.168.56.11
  user: ubuntu
  ssh:
    port: 22

# Inline (flow) style
server: { host: 192.168.56.11, user: ubuntu }
```

### Lists of Dictionaries (the Ansible Task Pattern)

```yaml
tasks:
  - name: Install nginx
    ansible.builtin.apt:
      name: nginx
      state: present

  - name: Start nginx
    ansible.builtin.service:
      name: nginx
      state: started
```

Each `- name:` begins a new dictionary inside the `tasks` list.

### Data Types

```yaml
string: hello
quoted_string: "port 80"
integer: 42
float: 3.14
boolean: true
null_value: null
```

### Multi-Line Strings

```yaml
# Literal block: newlines are preserved
message: |
  Line one
  Line two

# Folded block: newlines become spaces
description: >
  This long sentence is folded
  into a single line.
```

### Common YAML Pitfalls

| Mistake | Fix |
|---|---|
| Using tabs | Use spaces only. Configure your editor to insert spaces. |
| Missing space after colon (`key:value`) | Write `key: value`. |
| Inconsistent indentation | Keep sibling items aligned. |
| Value starts with `{{ }}` unquoted | Quote it: `path: "{{ my_var }}/file"`. |
| Value contains `: ` or `#` | Wrap the value in quotes. |
| File mode written as `0644` unquoted | Quote it (`"0644"`) so it isn't parsed as a number. |
| Ambiguous booleans (`yes`/`no`) | Prefer `true`/`false`. |

Validate before running:

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
yamllint playbook.yml
```

---

## 3. Variable Precedence

When the same variable is defined in several places, Ansible must decide which value wins. This is **variable precedence**: higher-precedence sources **override** lower ones.

### Simplified Hierarchy (Lowest to Highest)

```
1.  Role defaults               (roles/x/defaults/main.yml)    <- LOWEST
2.  Inventory group_vars        (group_vars/webservers.yml)
3.  Inventory host_vars         (host_vars/web-server-1.yml)
4.  Play vars                   (vars: in the playbook)
5.  Play vars_files             (vars_files: in the playbook)
6.  Role vars                   (roles/x/vars/main.yml)
7.  Task vars / block vars      (vars: on a task or block)
8.  set_fact / register         (set during the run)
9.  Extra vars                  (-e on the command line)       <- HIGHEST
```

> This is a condensed version. Ansible's official list has about 22 levels. Always check the official "Using variables: variable precedence" docs for the full order.

### The Golden Rules

- **Role `defaults/` are the lowest.** They are meant to be overridden by anyone.
- **Extra vars (`-e`) always win**, over everything including `set_fact` and role `vars/`.
- **More specific beats more general:** host-level beats group-level, and group-level beats `all`.
- **Role `vars/` is much stronger than `defaults/`.** Put tunable values in `defaults/`, and constants in `vars/`.

### Example: Same Variable, Three Levels

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
```

```yaml
# playbook.yml
- hosts: webservers
  vars:
    nginx_port: 8080       # overrides role default
  roles:
    - nginx
```

```bash
# Uses 8080 (play vars beat role defaults)
ansible-playbook -i inventory.ini playbook.yml

# Uses 9090 (extra vars beat everything)
ansible-playbook -i inventory.ini playbook.yml -e nginx_port=9090
```

Extra vars can also be loaded from a file:

```bash
ansible-playbook playbook.yml -e "@extra_vars.yml"
```

### `ansible.cfg`: Configuration Settings

`ansible.cfg` configures **Ansible's behavior** (inventory path, default SSH user, forks, and so on). It is not a place to define playbook variables, but it follows its own override order.

```ini
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
host_key_checking = False
forks = 10
```

**Which `ansible.cfg` is used** (the first one found wins, and files are not merged):

1. `ANSIBLE_CONFIG` environment variable (path to a file)
2. `./ansible.cfg` in the current directory
3. `~/.ansible.cfg` in your home directory
4. `/etc/ansible/ansible.cfg` (system-wide)

**Which setting value is used** (lowest to highest):

```
Built-in defaults  ->  ansible.cfg  ->  ANSIBLE_* environment variables  ->  command-line flags
```

For example, `ansible.cfg` may say `remote_user = ubuntu`, while `ansible all -m ping -u admin` uses `admin` for that run.

> **Gotcha:** A connection variable set in the inventory (such as `ansible_user=ubuntu`) overrides the `-u` flag. The `-u`/`--user` flag is treated as the lowest-priority source for the connection user. If a flag seems ignored, check the inventory.

### Debugging Precedence

```bash
# Show the value a host actually receives
ansible-inventory -i inventory.ini --host web-server-1

# Print a variable during a run
ansible webservers -i inventory.ini -m ansible.builtin.debug -a "var=nginx_port"

# Show effective config and where each setting came from
ansible-config dump --only-changed
```

---

## 4. Nginx & Three-Tier Architecture

### Three-Tier Architecture

A **three-tier application** separates a system into three independent layers, each of which can be deployed, scaled, and secured on its own.

| Tier | Role | Typical Technology |
|---|---|---|
| **1. Presentation (Web)** | What users see and interact with | HTML/CSS/JS, React, served by **Nginx** |
| **2. Application (Logic)** | Business logic, APIs, processing | Node.js, Python (Django/Flask), Java, Go |
| **3. Data (Database)** | Persistent storage | PostgreSQL, MySQL, MongoDB |

```
                    +-------------------+
   User  -------->  |   Nginx (Tier 1)  |   port 80 / 443  (public)
                    |   reverse proxy   |
                    +---------+---------+
                              |  proxy_pass
                              v
                    +-------------------+
                    |  App Server       |   port 3000  (private)
                    |  (Tier 2)         |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |  Database         |   port 5432  (private)
                    |  (Tier 3)         |
                    +-------------------+
```

**Benefits:** independent scaling, better security (the database is never exposed to the internet), easier maintenance, and clear separation of responsibilities.

### What Nginx Does

Nginx is a high-performance web server that can also act as a **reverse proxy**, **load balancer**, and **TLS terminator**.

### Reverse Proxy

A **forward proxy** sits in front of *clients* and hides them from servers. A **reverse proxy** sits in front of *servers* and hides them from clients.

- Users talk only to Nginx on ports 80/443.
- Nginx forwards requests to the correct backend service.
- Backends stay on private ports or private networks.

### Port Forwarding with `proxy_pass`

Nginx listens on one port and forwards traffic to a backend on another port:

```nginx
server {
    listen 80;                          # public-facing port
    server_name example.com;

    # Tier 1: serve static frontend files
    location / {
        root /var/www/html;
        index index.html;
    }

    # Tier 2: forward API calls to the backend app
    location /api/ {
        proxy_pass http://127.0.0.1:3000;   # backend port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

How it works: a request to `http://example.com/api/users` reaches Nginx on port 80, matches `location /api/`, and is forwarded to `127.0.0.1:3000`. The response comes back through Nginx to the user.

### Load Balancing Across Multiple Backends

```nginx
upstream backend_pool {
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}

server {
    listen 80;
    location /api/ {
        proxy_pass http://backend_pool;
    }
}
```

### Why Put Nginx in Front?

| Benefit | Explanation |
|---|---|
| **Single entry point** | One public port instead of exposing every service. |
| **Security** | Backends and databases stay hidden on private networks. |
| **TLS termination** | Nginx handles HTTPS so apps don't have to. |
| **Load balancing** | Spreads traffic across multiple app servers. |
| **Performance** | Fast static file serving and caching. |

Useful Nginx commands:

```bash
sudo nginx -t                    # test configuration syntax
sudo systemctl reload nginx      # apply config without dropping connections
sudo systemctl status nginx      # check service state
```

---

## 5. Practical Code Examples

### Ad-Hoc Commands

Ad-hoc commands run a single module against hosts without a playbook. General form:

```
ansible <host-pattern> -i <inventory> -m <module> -a "<arguments>" [options]
```

```bash
# Connectivity test (not ICMP; checks SSH + Python)
ansible webservers -i inventory.ini -m ping

# Run a shell command (default module is "command")
ansible webservers -i inventory.ini -a "uptime"
ansible webservers -i inventory.ini -a "df -h"

# Install a package (needs privilege escalation)
ansible webservers -i inventory.ini -m ansible.builtin.apt \
  -a "name=nginx state=present update_cache=yes" --become

# Start and enable a service
ansible webservers -i inventory.ini -m ansible.builtin.service \
  -a "name=nginx state=started enabled=yes" --become

# Copy a file to the servers
ansible webservers -i inventory.ini -m ansible.builtin.copy \
  -a "src=./index.html dest=/var/www/html/index.html mode=0644" --become

# Gather system facts
ansible web-server-1 -i inventory.ini -m ansible.builtin.setup

# Limit to one host
ansible webservers -i inventory.ini -a "uptime" --limit web-server-2
```

### Project Layout

```
nginx-project/
├── ansible.cfg
├── inventory.ini
├── site.yml
└── templates/
    └── nginx.conf.j2
```

### `ansible.cfg`

```ini
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
host_key_checking = False
```

### `inventory.ini`

```ini
[webservers]
web-server-1 ansible_host=192.168.56.11 ansible_user=ubuntu
web-server-2 ansible_host=192.168.56.12 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### `templates/nginx.conf.j2`

```jinja
server {
    listen {{ nginx_port }};
    server_name {{ inventory_hostname }};

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

### `site.yml`: Install, Configure, and Run Nginx

```yaml
---
- name: Install and configure Nginx
  hosts: webservers
  become: true

  vars:
    nginx_port: 80

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Deploy nginx site configuration from template
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        owner: root
        group: root
        mode: "0644"
      notify: Reload nginx

    - name: Deploy landing page
      ansible.builtin.copy:
        content: "<h1>Hello from {{ inventory_hostname }}</h1>\n"
        dest: /var/www/html/index.html
        mode: "0644"

    - name: Ensure nginx is started and enabled at boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

### Running the Playbook

```bash
# 1. Check YAML and playbook syntax
ansible-playbook site.yml --syntax-check

# 2. Dry run: show what would change, without changing anything
ansible-playbook site.yml --check --diff

# 3. Apply for real
ansible-playbook site.yml

# 4. Override a variable at runtime (extra vars win)
ansible-playbook site.yml -e nginx_port=8080

# 5. Verify
curl http://192.168.56.11
```

Run it twice. The second run should report `changed=0`, proving **idempotency**. Handlers only fire when the template task reports `changed`.

### Refactoring into a Role

```bash
mkdir roles && ansible-galaxy init roles/nginx
```

Move the pieces into the role:

| Original location | Role location |
|---|---|
| `tasks:` block | `roles/nginx/tasks/main.yml` |
| `handlers:` block | `roles/nginx/handlers/main.yml` |
| `templates/nginx.conf.j2` | `roles/nginx/templates/nginx.conf.j2` |
| `vars: nginx_port: 80` | `roles/nginx/defaults/main.yml` |

`roles/nginx/tasks/main.yml`:

```yaml
---
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Deploy nginx site configuration from template
  ansible.builtin.template:
    src: nginx.conf.j2          # resolved from the role's templates/ directory
    dest: /etc/nginx/sites-available/default
    mode: "0644"
  notify: Reload nginx

- name: Ensure nginx is started and enabled at boot
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

`roles/nginx/handlers/main.yml`:

```yaml
---
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

`roles/nginx/defaults/main.yml`:

```yaml
---
nginx_port: 80
```

The playbook becomes tiny:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - nginx
```

---

## 6. Mock Interview Questions

### Q1. What is the difference between an Ansible playbook and a role? When would you use a role?

**Answer:**
A playbook is a YAML file that maps groups of hosts to tasks. It is the entry point that says *what runs where*. A role is a standardized directory structure (`tasks/`, `handlers/`, `templates/`, `defaults/`, `vars/`, `files/`, `meta/`) that packages everything needed to configure one component, such as Nginx or MySQL.

Use roles once a playbook grows beyond a handful of tasks, or when you need reuse across projects and environments. Roles keep playbooks short and readable, make components testable in isolation, and can be shared through Git or Ansible Galaxy. A site playbook then simply lists roles: `roles: [nginx, app, monitoring]`.

---

### Q2. A variable `nginx_port` is set in the role's `defaults/main.yml` as 80, in the playbook's `vars` as 8080, and you run the playbook with `-e nginx_port=9090`. Which value is used, and why?

**Answer:**
**9090.** Extra vars passed with `-e` sit at the top of Ansible's variable precedence and override every other source. Without `-e`, the playbook's `vars` value (8080) would win, because play vars have higher precedence than role defaults. Role `defaults/` are the lowest-precedence location by design, so users can override them easily.

Two follow-up points: role `vars/main.yml` has much higher precedence than `defaults/`, so tunable settings belong in `defaults/`. And `ansible.cfg` does not hold playbook variables. It controls Ansible's behavior, and its settings are overridden by environment variables and then command-line flags. To debug, use `ansible-inventory --host <name>` or the `debug` module.

---

### Q3. What are handlers, and why use them instead of a normal task that restarts Nginx?

**Answer:**
A handler is a task that runs only when another task notifies it with `notify`, and only if that task reported `changed`. Handlers run once at the end of the play, even if notified multiple times.

A plain "restart nginx" task would run on every playbook execution, causing needless service interruptions and breaking the idempotent, `changed=0` behavior on repeated runs. With a handler, Nginx reloads only when its configuration file actually changed. This also guarantees the reload happens after all config tasks complete, and `reload` (instead of `restart`) applies changes without dropping active connections.

---

### Q4. Explain a three-tier architecture and how Nginx acts as a reverse proxy within it.

**Answer:**
A three-tier architecture splits an application into the **presentation tier** (frontend, served to users), the **application tier** (backend logic and APIs), and the **data tier** (database). Each tier scales and is secured independently, and only the first needs to be publicly reachable.

Nginx sits in the presentation tier as a **reverse proxy**. It listens on public ports 80/443, serves static frontend files, and uses `proxy_pass` to forward API requests (for example `/api/`) to the backend on a private port such as 3000. The client never talks to the backend or database directly. This gives a single entry point, hides internal topology, enables TLS termination in one place, and allows load balancing across several backend instances through an `upstream` block. This is **port forwarding at the HTTP layer**: public port 80 maps to an internal service port.

---

## 7. Hands-on Practice Assignment

**Goal:** Build a project that deploys Nginx with a playbook, then refactor it into a role and experiment with variable precedence.

### Prerequisites

- Ansible installed on your control node.
- Two Ubuntu servers reachable as `ubuntu` with your SSH key.
- Git installed (optional, for the final step).

### Step 1: Create the Project

```bash
mkdir ~/nginx-project && cd ~/nginx-project
git init
mkdir templates
```

### Step 2: Create `ansible.cfg` and `inventory.ini`

Use the files from [Section 5](#5-practical-code-examples), replacing the IPs with your own.

### Step 3: Verify Connectivity with Ad-Hoc Commands

```bash
ansible webservers -m ping
ansible webservers -a "uptime"
ansible webservers -m ansible.builtin.setup -a "filter=ansible_distribution*"
```

**Expected:** Both hosts return `SUCCESS`. Because `ansible.cfg` sets the inventory path, you no longer need `-i`.

### Step 4: Write the Template

Create `templates/nginx.conf.j2` with the content from Section 5.

### Step 5: Write the Playbook

Create `site.yml` with the install, template, copy, and service tasks plus the `Reload nginx` handler from Section 5.

### Step 6: Validate, Dry-Run, Apply

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
```

### Step 7: Prove Idempotency and Handler Behavior

```bash
ansible-playbook site.yml      # expect changed=0, and the handler does NOT run
```

Now change `nginx_port` in the playbook to `8080` and run again. The template task changes, and the **Reload nginx** handler fires.

### Step 8: Experiment with Variable Precedence

```bash
ansible-playbook site.yml                       # uses the play vars value
ansible-playbook site.yml -e nginx_port=9090    # extra vars win
curl http://192.168.56.11:9090
```

Then add a `group_vars/webservers.yml` file with a different `nginx_port`. Predict which value wins before you run it, then check with `ansible-inventory --host web-server-1`.

### Step 9: Refactor into a Role

Create `roles/nginx` with `ansible-galaxy init`, move tasks, handlers, template, and defaults into it, and reduce `site.yml` to:

```yaml
---
- hosts: webservers
  become: true
  roles:
    - nginx
```

Run it again. The result should be identical (`changed=0`).

### Step 10: Add a Reverse Proxy Location

Extend the template with a `location /api/` block that uses `proxy_pass http://127.0.0.1:3000;`. Start any simple service on port 3000 on one server (for example `python3 -m http.server 3000`), then request `http://<server-ip>/api/` and confirm Nginx forwards it.

### Step 11: Commit Your Work

```bash
echo "*.retry" > .gitignore
git add .
git commit -m "Nginx role and playbook"
```

### Stretch Goals

- Run `yamllint` and `ansible-lint` on your files and fix every warning.
- Store a secret with `ansible-vault encrypt_string` and use it in a playbook.
- Add a second role (for example `common`, which installs `curl` and `git`) and apply both roles.
- Add a `staging.ini` and `production.ini` inventory and reuse the same role for both.

---

## 8. Useful Links

| Resource | Link |
|---|---|
| Lecture 4: Ansible Playbooks & Roles: Nginx, Handlers & Templates (Picsart Academy) | [YouTube](https://youtu.be/pGspZwt_Qmk?si=wExUo7nEcifxtyN7) |
| Ansible documentation | [docs.ansible.com](https://docs.ansible.com) |
| Ansible Galaxy (roles and collections) | [galaxy.ansible.com](https://galaxy.ansible.com) |
| Nginx documentation | [nginx.org/en/docs](https://nginx.org/en/docs/) |
| Picsart Academy | [picsartacademy.am](https://picsartacademy.am/) |

---

## License

MIT, free to use, adapt, and share.

---

🎥 **Lecture:** [Watch the full lecture on YouTube](https://youtu.be/pGspZwt_Qmk?si=wExUo7nEcifxtyN7)