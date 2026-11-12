# Infrastructure as Code & Ansible Fundamentals

A practical summary of Infrastructure as Code (IaC) concepts, environment strategy, declarative vs. imperative tooling, and a hands-on introduction to Ansible inventories and ad-hoc commands.

## Table of Contents

1. [Infrastructure as Code (IaC) Fundamentals](#1-infrastructure-as-code-iac-fundamentals)
2. [Software Environments (Dev, Staging, Production)](#2-software-environments-dev-staging-production)
3. [Declarative vs. Imperative Approaches](#3-declarative-vs-imperative-approaches)
4. [Provisioning vs. Configuration Management Tools](#4-provisioning-vs-configuration-management-tools)
5. [Single Source of Truth (SSOT) & Configuration Drift](#5-single-source-of-truth-ssot--configuration-drift)
6. [Ansible Architecture, Inventory & Ad-Hoc Commands](#6-ansible-architecture-inventory--ad-hoc-commands)
7. [Mock Interview Questions](#7-mock-interview-questions)
8. [Hands-on Practice Assignment](#8-hands-on-practice-assignment)
9. [Useful Links](#9-useful-links)

---

## 1. Infrastructure as Code (IaC) Fundamentals

### Definition

**Infrastructure as Code (IaC)** is the practice of managing and provisioning infrastructure (servers, networks, load balancers, databases, firewall rules, OS packages, and services) through machine-readable definition files rather than manual processes or interactive configuration tools.

Instead of logging into a cloud console or SSH-ing into a server to click buttons and type commands, you *describe* your infrastructure in code (YAML, HCL, JSON, etc.) and let a tool create or modify it for you.

### Why IaC Is Crucial

| Benefit | Explanation |
|---|---|
| **Eliminates manual errors** | Humans mistype commands, skip steps, and forget what they changed. Code executes the same way every time. |
| **Repeatability** | The same definition can build one server or one thousand. Disaster recovery becomes "re-run the code." |
| **Version control (Git) integration** | Infrastructure changes are committed, diffed, reviewed via pull requests, and can be rolled back with `git revert`. |
| **Auditability** | `git log` and `git blame` show who changed what, when, and why. |
| **Speed** | Environments that took days to hand-build can be created in minutes. |
| **Documentation** | The code *is* the documentation of what your infrastructure looks like. |
| **Collaboration** | Teams review infrastructure changes with the same workflow as application code (PRs, CI checks, approvals). |

---

## 2. Software Environments (Dev, Staging, Production)

| Environment | Purpose | Typical Audience | Characteristics |
|---|---|---|---|
| **Development (Dev)** | Where developers write and test code as they build features. | Developers | Frequently changing, unstable, small scale, often disposable. |
| **Staging** | A pre-production replica used for final validation: integration, performance, and user acceptance testing. | QA, product owners, DevOps | Should mirror production as closely as possible. |
| **Production (Prod)** | The live system serving real users. | End users | Stable, monitored, secured, high availability, strict change control. |

### Typical Promotion Flow

```
Developer commit -> Dev -> Staging -> Production
```

### How IaC Guarantees Identical Environments

The classic failure: *"It worked on my machine!"* This happens because environments **drift apart** (different package versions, OS patches, config files, or environment settings).

With IaC:

- All environments are built from the **same code**.
- Only **variables** differ (instance size, replica count, domain names, secrets).
- Because the build process is automated and deterministic, structure and software configuration are identical across Dev, Staging, and Prod.

```
            +-----------------------+
            |  Same IaC Codebase    |
            +-----------+-----------+
                        |
      +-----------------+-----------------+
      |                 |                 |
  vars: dev        vars: staging      vars: prod
  (1 small VM)     (2 medium VMs)     (6 large VMs)
```

> Note: "100% identical" refers to the **configuration and software stack**. Scale (CPU, node count) and secrets typically differ intentionally via variables.

---

## 3. Declarative vs. Imperative Approaches

### Imperative ("HOW")

You write the **exact sequence of steps** to reach a goal. You are responsible for handling every condition: "if it's already installed, skip," "if the service is running, don't restart," and so on.

- Examples: Bash scripts, Python scripts, raw CLI command sequences.
- Pros: Full control, flexible for odd one-off tasks.
- Cons: Not inherently safe to re-run; error handling and state checks are manual; scripts grow complex quickly.

### Declarative ("WHAT")

You describe the **desired end state**, and the tool figures out how to get there.

- Examples: Ansible (when using modules), Terraform/OpenTofu, Puppet, Kubernetes manifests.
- Pros: Readable, safe to re-run, self-documenting.
- Cons: Less flexibility for highly custom procedural logic.

### Idempotency

**Idempotency** means running the same operation multiple times produces the **same result** as running it once. If the system is already in the desired state, the tool makes **no changes**.

- First run: Nginx is missing -> installed -> reports `changed`.
- Second run: Nginx is already present -> nothing to do -> reports `ok`.

This is why declarative tools can be run continuously (e.g., on a schedule or in CI/CD) without fear of breaking things.

### Imperative Example: Managing Nginx with Bash

Every check has to be written by hand:

```bash
#!/bin/bash
# install_nginx.sh - the HOW: we must handle every check ourselves

# Check if nginx is installed
if ! dpkg -s nginx >/dev/null 2>&1; then
    echo "Installing nginx..."
    apt-get update
    apt-get install -y nginx
else
    echo "nginx already installed"
fi

# Check if nginx is running
if ! systemctl is-active --quiet nginx; then
    echo "Starting nginx..."
    systemctl start nginx
fi

# Check if nginx is enabled at boot
if ! systemctl is-enabled --quiet nginx; then
    echo "Enabling nginx..."
    systemctl enable nginx
fi
```

A declarative tool expresses the same intent as a single desired state ("nginx is present, running, and enabled") and handles these checks internally.

---

## 4. Provisioning vs. Configuration Management Tools

Infrastructure automation has two distinct jobs, and different tools specialize in each.

### Provisioning Tools: Terraform / OpenTofu

**Job:** Create, modify, and destroy the **infrastructure itself**: VMs, VPCs, subnets, load balancers, databases, DNS records, IAM roles.

> **Analogy:** The **construction crew** that builds the empty house: foundation, walls, plumbing, and electrical wiring.

- Declarative, with a **state file** tracking what exists.
- Uses providers (AWS, Azure, GCP, etc.) to talk to APIs.
- OpenTofu is the open-source fork of Terraform, managed by the Linux Foundation.

### Configuration Management Tools: Ansible / Puppet (also Chef, Salt)

**Job:** Configure what runs **inside** the servers: install packages, manage config files, start services, create users, apply patches.

> **Analogy:** The **interior designer and movers** who furnish the house: installing appliances, arranging furniture, and setting up utilities.

| Tool | Model | Agent? |
|---|---|---|
| **Ansible** | Push (control node pushes changes) | Agentless (SSH/WinRM) |
| **Puppet** | Pull (nodes poll a master) | Agent-based |

### Division of Labor

```
 Terraform / OpenTofu                 Ansible / Puppet
 --------------------                 ----------------
 "Create 3 Ubuntu VMs         -->     "Install Nginx, deploy app,
  in a private subnet with            configure TLS, create users,
  a load balancer."                   harden SSH."
```

Common pattern: Terraform provisions servers and outputs their IPs, then Ansible consumes those IPs as its inventory and configures them.

---

## 5. Single Source of Truth (SSOT) & Configuration Drift

### Single Source of Truth (SSOT)

The **SSOT** is the one authoritative location where the desired state of your infrastructure is defined. In modern IaC practice, this is a **Git repository**.

- If it isn't in Git, it isn't "real."
- Every change flows through commit -> pull request -> review -> merge -> automated apply.
- Production reflects what's in the `main` branch, not what someone remembers doing last Tuesday.

```
   Git Repository (SSOT)
          |
   Pull Request + Review
          |
   CI/CD Pipeline
          |
   Apply to servers
```

### Configuration Drift

**Configuration drift** occurs when the actual state of a server diverges from its intended, documented state, usually due to **manual changes**.

Example: An engineer SSH-es into a production server at 2 AM and edits `/etc/nginx/nginx.conf` to fix an incident. They forget to record it. Now:

- Production differs from Staging.
- The next "clean" deployment may overwrite the fix or break something.
- Nobody knows why the server behaves differently.

### How SSOT Prevents Drift

1. **All changes go through Git**, so the desired state is always recorded.
2. **Re-running idempotent IaC** automatically corrects out-of-band manual edits back to the declared state.
3. **Drift detection** (e.g., `terraform plan`) reveals where reality differs from code.
4. **Restricted manual access** (and treating servers as replaceable) discourages "snowflake" servers.

---

## 6. Ansible Architecture, Inventory & Ad-Hoc Commands

### Agentless Architecture

Ansible requires **no agent software** on managed nodes.

- **Control node:** The machine where Ansible is installed and run (Linux/macOS/WSL).
- **Managed nodes:** The target servers being configured.
- **Transport:**
  - **SSH** for Linux/Unix hosts.
  - **WinRM** (or SSH) for Windows hosts.
- **Mechanism:** Ansible connects, copies a small Python module to a temp directory on the target, executes it, collects the JSON result, and cleans up.
- **Requirements on managed nodes:** SSH access and Python (for most modules).

```
+----------------+     SSH (22)      +---------------+
|  Control Node  | ----------------> |  web-server-1 |
|  (Ansible)     |                   +---------------+
|                |     SSH (22)      +---------------+
|                | ----------------> |  web-server-2 |
+----------------+                   +---------------+
```

**Benefits:** No agents to install, upgrade, or secure; smaller attack surface; easy onboarding of new hosts.

### The Inventory File (`inventory.ini`)

An inventory file tells Ansible **which machines to manage** and **how to connect to them**. Because Ansible has no agents and no built-in list of servers, the inventory is its address book. The `.ini` format is one of the simplest ways to write it (YAML is the other common option).

```ini
[webservers]
web-server-1 ansible_host=192.168.56.11 ansible_user=ubuntu
web-server-2 ansible_host=192.168.56.12 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

#### Line-by-Line Breakdown

| Part | Meaning |
|---|---|
| `[webservers]` | A **group header**. Everything beneath it belongs to the group `webservers` until the next header. |
| `web-server-1` | The **inventory alias**: a custom name you choose. It appears in Ansible's output and is what you target with `--limit`. It doesn't need to resolve in DNS. |
| `ansible_host=192.168.56.11` | The **real IP address** (or hostname) Ansible connects to. Without it, Ansible tries to resolve the alias as a hostname. |
| `ansible_user=ubuntu` | The **SSH login user**. `ubuntu` is the default on Ubuntu cloud images. |
| `[webservers:vars]` | A **group variables** section. The `:vars` suffix means "apply these to every host in `webservers`." |
| `ansible_ssh_private_key_file` | Path to the SSH private key used for authentication. |
| `ansible_python_interpreter` | Which Python to use on the target; avoids interpreter-discovery warnings. |

#### Why Aliases and `ansible_host` Are Separate

Separating the name from the address gives you readable names and flexible addressing. If a server's IP changes, you edit one line and everything that references `web-server-1` keeps working.

#### Host-Level vs. Group-Level Variables

```ini
# Host-level: applies only to this host (on the same line)
web-server-1 ansible_host=192.168.56.11 ansible_user=ubuntu

# Group-level: applies to every host in the group
[webservers:vars]
ansible_user=ubuntu
```

If both define the same variable, the **host-level value wins**. Moving shared values like `ansible_user=ubuntu` into `:vars` removes repetition:

```ini
[webservers]
web-server-1 ansible_host=192.168.56.11
web-server-2 ansible_host=192.168.56.12

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

#### Common Connection Variables

| Variable | Purpose |
|---|---|
| `ansible_host` | IP or DNS name to connect to |
| `ansible_user` | SSH username |
| `ansible_port` | SSH port (default `22`) |
| `ansible_ssh_private_key_file` | Path to private key |
| `ansible_connection` | Transport type: `ssh`, `winrm`, `local` |
| `ansible_python_interpreter` | Python path on the target |
| `ansible_become` | Whether to escalate privileges (sudo) |

#### Multiple Groups and Nested Groups

A host can belong to several groups, and groups can contain other groups using `:children`:

```ini
[webservers]
web-server-1 ansible_host=192.168.56.11
web-server-2 ansible_host=192.168.56.12

[dbservers]
db-server-1 ansible_host=192.168.56.21

[production:children]
webservers
dbservers

[all:vars]
ansible_user=ubuntu
```

- `production` contains every host from `webservers` and `dbservers`.
- `all` is a built-in group containing every host, so `[all:vars]` applies to everything.
- `ungrouped` is another built-in group for hosts not listed under any group.

#### Host Ranges

```ini
[webservers]
web-server-[1:5] ansible_user=ubuntu
```

This expands to `web-server-1` through `web-server-5`. Since no `ansible_host` is set, DNS or `/etc/hosts` entries must exist for those names.

#### Targeting Hosts

Point Ansible at the file with `-i`, then pick a target by group name, host alias, or `all`:

```bash
# Target a group
ansible webservers -i inventory.ini -m ping

# Target a single host by alias
ansible web-server-1 -i inventory.ini -m ping

# Target every host
ansible all -i inventory.ini -m ping

# Target one host within a group
ansible webservers -i inventory.ini -m ping --limit web-server-2
```

#### Inspecting and Validating the Inventory

```bash
# List groups and hosts as a tree
ansible-inventory -i inventory.ini --graph

# Show all hosts with their resolved variables (JSON)
ansible-inventory -i inventory.ini --list

# Show variables for one host
ansible-inventory -i inventory.ini --host web-server-1
```

Example `--graph` output:

```
@all:
  |--@ungrouped:
  |--@webservers:
  |  |--web-server-1
  |  |--web-server-2
```

#### Common Mistakes

- **Spaces around `=`:** write `ansible_user=ubuntu`, not `ansible_user = ubuntu`.
- **Missing `ansible_host`:** Ansible tries to resolve the alias as a hostname and fails with "Could not resolve hostname."
- **Wrong key permissions:** the private key must be `chmod 600`, or SSH refuses it.
- **Quoting values with spaces:** use quotes, e.g. `ansible_ssh_common_args='-o StrictHostKeyChecking=no'`.
- **Group names with hyphens:** use underscores (`web_servers`), since hyphens trigger a warning in newer Ansible versions. Host aliases with hyphens are fine.

#### Key Takeaway

`inventory.ini` answers three questions for Ansible: **who** are the targets (hosts and aliases), **how are they organized** (groups), and **how do I reach them** (connection variables like `ansible_host` and `ansible_user`).

### Ad-Hoc Command: Ping Module

Ad-hoc commands run a single module against hosts without writing a playbook.

```bash
ansible webservers -i inventory.ini -m ping
```

Expected output:

```
web-server-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
web-server-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> **Important:** The Ansible `ping` module is **not** an ICMP ping. It verifies that Ansible can log in over SSH and execute Python on the target.

Other useful ad-hoc patterns:

```bash
# Run a shell command on all hosts
ansible all -i inventory.ini -a "uptime"

# Install a package with privilege escalation
ansible webservers -i inventory.ini -m ansible.builtin.apt -a "name=htop state=present" --become
```

---

## 7. Mock Interview Questions

### Q1. Explain the difference between declarative and imperative approaches. Why does idempotency matter?

**Answer:**
An imperative approach specifies *how* to reach a result through an explicit sequence of commands (for example, a Bash script that runs `apt install`, then `systemctl start`). The author must handle edge cases such as "already installed" or "already running." A declarative approach specifies *what* the end state should be (for example, "nginx is present and running"), and the tool determines which actions, if any, are needed.

Idempotency means applying the same configuration repeatedly yields the same state without unintended side effects. It matters because it makes automation **safe to re-run**, supports drift correction, enables CI/CD pipelines that apply config continuously, and makes failures recoverable (just run it again). Most Ansible modules (`apt`, `service`, `copy`, `template`) are idempotent, though `command` and `shell` are not unless guarded with `creates`, `removes`, or `changed_when`.

---

### Q2. Why is Ansible called "agentless," and what are the trade-offs?

**Answer:**
Ansible doesn't require a persistent daemon or agent on managed nodes. The control node connects via **SSH** (Linux) or **WinRM** (Windows), transfers small module scripts, executes them, returns JSON results, and removes the temporary files.

**Advantages:** Nothing to install/maintain on targets, reduced attack surface, quick bootstrapping, works with network devices and cloud APIs.
**Trade-offs:** It is *push-based*, so changes only apply when you run it (no continuous self-healing unless scheduled via cron, AWX/AAP, or CI); SSH connection overhead can slow very large fleets (mitigated with `forks`, pipelining, and ControlPersist); and most modules require Python on the target.

---

### Q3. What is the difference between Terraform and Ansible? Would you use both?

**Answer:**
Terraform (or OpenTofu) is a **provisioning** tool: it creates and manages infrastructure resources (VMs, networks, load balancers, databases) through provider APIs, tracks them in a **state file**, and is fully declarative with a plan/apply workflow. Ansible is primarily a **configuration management** tool: it configures the OS and software *inside* those resources (packages, files, services, users).

They're complementary. A common workflow: Terraform provisions VMs and outputs their IPs -> those IPs feed an Ansible inventory (static or dynamic) -> Ansible installs and configures the application stack. Ansible *can* provision cloud resources and Terraform *can* run provisioners, but each is strongest in its own domain. Terraform's state tracking makes it better for lifecycle management (create/update/destroy), while Ansible's procedural-friendly task flow is better for in-server configuration and orchestration.

---

### Q4. A teammate says, "I ran `ansible all -m ping` and got `pong`, so the servers are reachable via ICMP." Is that correct?

**Answer:**
No. **ICMP** (Internet Control Message Protocol) operates at the network layer (Layer 3); the traditional `ping` utility sends ICMP Echo Request packets and waits for Echo Reply. It confirms basic IP reachability only and is often blocked by firewalls or cloud security groups.

Ansible's `ping` module is an **application-level check**: it confirms that (1) Ansible can reach the host over TCP (SSH on port 22 or WinRM), (2) authentication succeeds, (3) privilege and Python interpreter requirements are met, and (4) the module executed and returned `pong`. A host can fail ICMP ping (blocked) but succeed with Ansible ping, or respond to ICMP but fail Ansible ping (wrong key, SSH down, no Python). For network-level troubleshooting you'd use `ping`, `traceroute`, or `nc -zv host 22`; for automation readiness, use `ansible -m ping`.

---

## 8. Hands-on Practice Assignment

**Goal:** Build an Ansible inventory, validate it, verify connectivity, and run ad-hoc commands.

### Prerequisites

- Ansible installed on your control node (`pip install ansible` or `sudo apt install ansible`).
- Two Ubuntu servers (VMs, cloud instances, or containers with SSH) reachable with the `ubuntu` user and your SSH key.

### Step 1: Create the Project Directory

```bash
mkdir ~/ansible-practice && cd ~/ansible-practice
```

### Step 2: Create the Inventory File

Create `inventory.ini` (replace the IPs with your own):

```ini
[webservers]
web-server-1 ansible_host=192.168.56.11 ansible_user=ubuntu
web-server-2 ansible_host=192.168.56.12 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Step 3: Validate the Inventory

```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host web-server-1
```

**Expected:** Both hosts appear under `@webservers`, and the second command shows `ansible_host` and `ansible_user` for `web-server-1`.

### Step 4: Verify Connectivity

```bash
ansible webservers -i inventory.ini -m ping
```

**Expected:** Both hosts return `SUCCESS` with `"ping": "pong"`. If not, check SSH key permissions (`chmod 600`), IP addresses, and that Python is installed on the targets.

### Step 5: Run an Ad-Hoc Uptime Command

```bash
ansible webservers -i inventory.ini -a "uptime"
```

**Expected:** Each server prints its current uptime and load average.

### Step 6: Target Hosts Selectively

```bash
ansible web-server-1 -i inventory.ini -a "uptime"
ansible webservers -i inventory.ini -a "uptime" --limit web-server-2
```

**Expected:** Only the selected host responds each time.

### Stretch Goals

- Move `ansible_user=ubuntu` from the host lines into `[webservers:vars]` and confirm everything still works.
- Add a `[dbservers]` group and a `[production:children]` group, then run `--graph` again.
- Commit the inventory to a Git repository to establish your **Single Source of Truth**.
- Create separate `staging.ini` and `production.ini` inventories with different IPs.

---

## 9. Useful Links

| Resource | Link |
|---|---|
| Lecture video | [YouTube](https://www.youtube.com/watch?v=a1KiHKXa7pA&list=PLG8Vnz_p0gOU&index=3) |

---

## License

MIT, free to use, adapt, and share.
