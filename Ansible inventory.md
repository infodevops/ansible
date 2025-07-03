---

# 📘 Ansible Inventory Configuration Guide

## 🎯 Introduction

We will now look at **configuring inventory in Ansible**.

Ansible can work with one or multiple systems in your infrastructure at the same time.
To manage remote systems, Ansible establishes **connectivity** to those servers using:

* **SSH** (for Linux)
* **WinRM (Windows Remote Management)** (for Windows)

> ✅ This is what makes **Ansible agentless**.

---

## 🧩 What Does Agentless Mean?

> ❗ Agentless means:

* No additional software (agent) is required on the target machines.
* A simple SSH or WinRM connection is enough.

In contrast, other tools often require agents to be installed and configured before any automation can be done.

---

## 📁 The Inventory File

Ansible gets information about the target systems from an **inventory file**.

### 📌 Default Inventory Location:

```
/etc/ansible/hosts
```

You can create your own inventory file or use the default one.

---

## 🗂️ INI-Style Inventory Format

An inventory file is typically in a format similar to `.ini`.

### 👇 Example:

```ini
server1.example.com
server2.example.com
```

### 📦 Grouping Servers

You can group servers by placing them under a **group name** in square brackets:

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com
```

> ✅ You can have **multiple groups** in one inventory file.

---

## 🏷️ Using Aliases and Inventory Parameters

You can define **aliases** and assign real host addresses using `ansible_host`.

### 👇 Example with Aliases and Parameters:

```ini
[webservers]
web1 ansible_host=192.168.1.10

[dbservers]
db1 ansible_host=db.example.com
```

Here:

* `web1` and `db1` are **aliases**
* `ansible_host` is the **real IP or FQDN**

---

## ⚙️ Common Inventory Parameters

| Parameter            | Purpose                                     |
| -------------------- | ------------------------------------------- |
| `ansible_host`       | IP/FQDN of the server                       |
| `ansible_connection` | Connection type: `ssh`, `winrm`, or `local` |
| `ansible_port`       | Port to connect to (default: `22` for SSH)  |
| `ansible_user`       | Username to connect as                      |
| `ansible_ssh_pass`   | SSH password for the user                   |

### 🔍 Example:

```ini
[webservers]
web1 ansible_host=192.168.1.10 ansible_user=ubuntu ansible_ssh_pass=password ansible_port=22
```

---

## 🧠 Understanding `ansible_connection`

This defines **how Ansible connects**:

| Value   | Use Case                   |
| ------- | -------------------------- |
| `ssh`   | Linux (default)            |
| `winrm` | Windows                    |
| `local` | Run tasks on local machine |

### 👇 Example:

```ini
localhost ansible_connection=local
```

---

## 🔐 Passwords and Security

> ⚠️ It's not ideal to store passwords like this in plain text.

### Recommended Best Practice:

* Use **SSH key-based authentication** for Linux.
* Use **Ansible Vault** to encrypt secrets.

But for **learning and testing**, simple username + password is fine.

---

## 🧪 Sample Inventory File (All Features Together)

```ini
[webservers]
web1 ansible_host=192.168.0.101 ansible_user=ubuntu ansible_port=22 ansible_ssh_pass=ubuntu123
web2 ansible_host=192.168.0.102 ansible_user=ubuntu ansible_connection=ssh

[dbservers]
db1 ansible_host=db.internal.local ansible_user=root ansible_ssh_pass=rootpass

[localhost]
127.0.0.1 ansible_connection=local
```

---

## 🧑‍💻 Try It Yourself

Head over to your Ansible project and:

1. Create a file:

   ```bash
   touch inventory.ini
   ```

2. Paste one of the above examples into it.

3. Run a test command:

   ```bash
   ansible all -i inventory.ini -m ping
   ```

---

## ✅ Summary

| Concept    | Description                                  |
| ---------- | -------------------------------------------- |
| Agentless  | No software installed on targets             |
| Inventory  | File that lists servers and groups           |
| Grouping   | Use `[groupname]` to define logical sets     |
| Aliases    | Define nicknames for IP/FQDN                 |
| Parameters | Customize connection: host, user, port, etc. |
| Security   | Use SSH keys or Vault for real setups        |

---
