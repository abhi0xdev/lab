Here are **mid-level interview notes for Ansible**—focused on real-world DevOps usage, troubleshooting, and decision-making.

---

# 🔹 1. Core Concepts (Mid-Level Focus)

### ✅ What is Ansible?

* **Definition:** An open-source **configuration management, automation, and orchestration tool**.
* **How it works:** Uses **SSH (agentless)** to connect to managed nodes and execute tasks defined in YAML (playbooks).
* **Why it works:** Push-based model → controller sends instructions → no agent overhead.

---

### ✅ Architecture

* **Control Node**

  * Machine where Ansible is installed
  * Executes playbooks

* **Managed Nodes**

  * Target systems (servers, VMs, containers)

* **Inventory**

  * List of hosts/groups (static or dynamic)

* **Modules**

  * Units of work (e.g., install package, copy file)

* **Playbooks**

  * YAML files describing automation tasks

* **Plugins**

  * Extend functionality (connection, lookup, callback)

---

### ✅ Idempotency

* Running same playbook multiple times → **same result**
* Achieved via modules checking state before applying changes

**Example:**

* `yum install nginx` runs only if nginx not installed

---

### ✅ Push vs Pull Model

* **Ansible = Push-based**
* No agent required → uses SSH
* Faster setup, but requires network access

---

### ✅ Tasks, Plays, Roles

* **Task:** Single action
* **Play:** Group of tasks applied to hosts
* **Role:** Reusable structured collection (tasks, vars, handlers)

---

### ✅ Handlers

* Triggered only when notified (on change)
* Used for actions like restarting services

---

### Constraints & Dependencies

* Requires:

  * SSH access
  * Python on target nodes
* Not ideal for:

  * Extremely large-scale real-time orchestration (compared to event-driven tools)

---

# 🔹 2. Key Terminology & Definitions

* **Inventory:** List of hosts and groups
* **Playbook:** YAML file defining automation steps
* **Module:** Pre-built unit of work
* **Role:** Structured reusable automation unit
* **Handler:** Triggered task (runs on change)
* **Facts:** System information gathered automatically
* **Variables:** Dynamic values used in playbooks
* **Template:** Jinja2-based dynamic file
* **Vault:** Encrypt sensitive data
* **Ad-hoc command:** One-liner execution without playbook

---

# 🔹 3. Tools, Frameworks, and Technologies

### 🔧 Ansible Ecosystem

* **Ansible Core**

  * CLI-based automation
  * Lightweight

* **Ansible Tower / AWX**

  * UI + RBAC + scheduling
  * Enterprise-grade automation

* **Ansible Galaxy**

  * Pre-built roles sharing platform

---

### 🔧 When to Use Ansible

* Configuration management
* Application deployment
* CI/CD automation
* Infrastructure provisioning (with cloud modules)

---

### Pros / Cons

**Pros**

* Agentless
* Simple YAML syntax
* Fast learning curve

**Cons**

* Slower at very large scale
* Limited state tracking vs Puppet

---

# 🔹 4. Comprehensive Interview Questions & Answers

---

## 🔸 Q1: How does Ansible work internally?

### ✅ Answer

* Control node reads playbook
* Connects via SSH to managed nodes
* Pushes modules to nodes
* Executes modules → returns results → removes modules

**Mid-level depth:**

* Explain execution flow + temporary module execution
* Mention Python dependency

---

## 🔸 Q2: Difference between Playbook and Ad-hoc commands?

| Feature     | Playbook          | Ad-hoc      |
| ----------- | ----------------- | ----------- |
| Use case    | Complex workflows | Quick tasks |
| Format      | YAML              | CLI         |
| Reusability | High              | Low         |

---

## 🔸 Q3: What is idempotency? Why important?

### ✅ Answer

* Ensures predictable infrastructure
* Avoids repeated changes
* Helps in safe re-runs (critical in CI/CD)

---

## 🔸 Q4: Scenario – Service not restarting after config change

### ✅ Answer

* Use handler:

```yaml
tasks:
  - name: Update config
    template:
      src: app.conf.j2
      dest: /etc/app.conf
    notify: Restart service

handlers:
  - name: Restart service
    service:
      name: app
      state: restarted
```

---

## 🔸 Q5: How do you manage environment-specific configs?

### ✅ Answer

* Use:

  * group_vars / host_vars
  * inventory-based separation
  * variable precedence

---

## 🔸 Q6: How do you secure secrets?

### ✅ Answer

* Use **Ansible Vault**
* Encrypt passwords, API keys

---

## 🔸 Q7: Scenario – Playbook slow for 500+ servers

### ✅ Answer

* Increase forks:

  ```
  ansible.cfg → forks=50
  ```
* Use:

  * async tasks
  * strategy = free
  * reduce fact gathering

---

## 🔸 Q8: How do you debug a failed playbook?

### ✅ Answer

* Use:

  ```
  -vvv
  ```
* Check:

  * logs
  * module output
  * SSH connectivity

---

## 🔸 Q9: What are roles and why important?

### ✅ Answer

* Modular, reusable structure
* Standard directory layout
* Improves maintainability

---

## 🔸 Q10: How do you integrate Ansible in CI/CD?

### ✅ Answer

* Use GitHub Actions / Jenkins
* Trigger playbooks on:

  * commit
  * deployment stage
* Example:

  * build → deploy via Ansible → restart service

---

### 🔥 Junior vs Mid vs Senior

* **Junior:** Defines Ansible basics
* **Mid:** Explains real scenarios + debugging
* **Senior:** Talks about scaling, architecture, trade-offs

---

# 🔹 5. Code Snippets / Examples

### ✅ Install Nginx

```yaml
- hosts: web
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

---

### ✅ Template Example

```yaml
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/app.conf
```

---

### Common Mistakes

* Hardcoding values
* Not using roles
* Ignoring idempotency

---

# 🔹 6. Practical & Real-World Scenarios

### 🚀 Scenario 1: Deploy Microservice on Kubernetes Nodes

* Install Docker
* Configure kubelet
* Pull images
* Restart services

---

### 🚀 Scenario 2: CI/CD Integration

* Git push → GitHub Actions → Ansible playbook → deploy app

---

### 🚀 Scenario 3: Config Drift Fix

* Re-run playbook → ensures consistency

---

# 🔹 7. Performance & Optimization

* Use **forks**
* Disable fact gathering:

```yaml
gather_facts: no
```

* Use **async tasks**
* Use **roles for reuse**

---

# 🔹 8. Debugging & Troubleshooting

### Common Issues

* SSH failure
* Permission denied
* Python not installed
* Module failure

### Steps

1. Run with `-vvv`
2. Check inventory
3. Test SSH manually
4. Validate YAML

---

# 🔹 9. Security / Reliability

* Use Vault for secrets
* Restrict SSH access
* Use RBAC (Tower)
* Avoid plaintext credentials

---


# 🔹 10. Flashcards

* **Idempotency?** → Same result every run
* **Inventory?** → Hosts list
* **Role?** → Reusable structure
* **Handler?** → Runs on change
* **Vault?** → Encrypt secrets

---