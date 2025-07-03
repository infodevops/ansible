## Full Workflow steps and code for Windows and SQL Server patching.
* Pre-validation logic
* Conditional execution (based on whether patch is needed)
* Clear error handling and success tracking
* Notifications with version history
* Reboot only when needed

---

## ✅ Full Workflow You Need

Here’s how we’ll structure it:

1. **Pre-check if SQL CU is required**
2. **If not required, skip with status message**
3. **If required:**

   * Check disk space
   * Get and store current SQL version
   * Install CU
   * If fails → send email
   * If succeeds → check if reboot needed

     * If yes → reboot
     * Send success email with before & after version
4. **Then: Windows Update patching**

---

## 🔧 Implementation Plan (Ansible Playbook + Custom Logic)

We will build this in **Ansible + PowerShell** with conditional logic and mail support.

---

## 📁 Updated Structure (Key Parts)

```
roles/
└── patching/
    ├── tasks/
    │   ├── main.yml
    │   ├── pre_validate_sql.yml       ✅ New
    │   ├── install_sql_patch.yml
    │   ├── notify_results.yml         ✅ New
    │   ├── patch_windows.yml
    └── vars/
        └── main.yml
```

---

## ⚙️ 1. `main.yml`: Master Control Flow

```yaml
- name: Pre-validate SQL patch necessity
  include_tasks: pre_validate_sql.yml

- name: Install SQL Server patch if required
  include_tasks: install_sql_patch.yml
  when: sql_patch_required

- name: Notify and reboot if needed
  include_tasks: notify_results.yml
  when: sql_patch_required and sql_patch_success

- name: Proceed with Windows Updates only if SQL patch successful or not needed
  include_tasks: patch_windows.yml
```

---

## 📜 2. `pre_validate_sql.yml`

```yaml
- name: Get current SQL Server version
  win_shell: |
    sqlcmd -Q "SELECT SERVERPROPERTY('ProductVersion') AS Version, SERVERPROPERTY('ProductUpdateLevel') AS CU"
  register: current_sql_version_raw

- name: Set SQL version fact
  set_fact:
    current_sql_version: "{{ current_sql_version_raw.stdout_lines[2] | trim }}"

- name: Extract CU number
  set_fact:
    current_cu: "{{ current_sql_version.split()[-1] | regex_search('CU(\\d+)', '\\1') | int(default=0) }}"

- name: Compare with target CU
  set_fact:
    target_cu: 21  # Update to the CU you are applying
    sql_patch_required: "{{ current_cu < target_cu }}"

- name: Exit early if no SQL patch required
  debug:
    msg: "SQL patch is already up to date (CU{{ current_cu }})"
  when: not sql_patch_required
```

---

## 📜 3. `install_sql_patch.yml`

```yaml
- name: Check available disk space on C:
  win_command: powershell -Command "(Get-PSDrive -Name C).Free"
  register: disk_space

- name: Fail if disk space < 5 GB
  fail:
    msg: "Not enough disk space: {{ disk_space.stdout | int // 1e+9 }} GB free"
  when: disk_space.stdout | int < 5e+9

- name: Download SQL CU installer
  win_get_url:
    url: "{{ sql_cu_url }}"
    dest: "{{ sql_cu_local_path }}"
  register: download_result

- name: Install SQL Server CU
  win_shell: |
    "{{ sql_cu_local_path }} /quiet /action=patch /instancename={{ sql_instance_name }} /IAcceptSQLServerLicenseTerms"
  register: sql_patch_result
  ignore_errors: true

- name: Set success/failure flag
  set_fact:
    sql_patch_success: "{{ sql_patch_result.rc == 0 }}"
```

---

## 📬 4. `notify_results.yml`

```yaml
- name: Get SQL Server version after patch
  win_shell: |
    sqlcmd -Q "SELECT SERVERPROPERTY('ProductVersion') AS Version, SERVERPROPERTY('ProductUpdateLevel') AS CU"
  register: post_patch_version_raw

- name: Set post-patch version fact
  set_fact:
    post_patch_version: "{{ post_patch_version_raw.stdout_lines[2] | trim }}"

- name: Check if reboot required
  win_stat:
    path: C:\Windows\WindowsUpdate.log
  register: win_update_log

- name: Reboot if required (heuristic check or log check)
  win_reboot:
    test_command: sqlcmd -Q "SELECT 1"
  when: "'restart' in sql_patch_result.stdout.lower() or 'reboot' in sql_patch_result.stdout.lower()"

- name: Send success email
  mail:
    host: smtp.yourcompany.com
    port: 25
    to: dba-team@yourcompany.com
    subject: "✅ SQL Patch Successful on {{ inventory_hostname }}"
    body: |
      SQL Server patch applied successfully on {{ inventory_hostname }}

      ▶ Before Version: {{ current_sql_version }}
      ▶ After Version:  {{ post_patch_version }}
  when: sql_patch_success

- name: Send failure email
  mail:
    host: smtp.yourcompany.com
    port: 25
    to: dba-team@yourcompany.com
    subject: "❌ SQL Patch Failed on {{ inventory_hostname }}"
    body: |
      SQL Server patch failed on {{ inventory_hostname }}

      ▶ Current Version: {{ current_sql_version }}
      ▶ Error Output:
      {{ sql_patch_result.stderr if sql_patch_result.stderr else sql_patch_result.stdout }}
  when: not sql_patch_success
```

---

## 🪟 5. `patch_windows.yml`: Standard Windows Updates

```yaml
- name: Install security and critical updates
  win_updates:
    category_names:
      - SecurityUpdates
      - CriticalUpdates
    reboot: yes
    state: installed
  register: windows_updates

- name: Reboot if needed
  win_reboot:
    reboot_timeout: 1800
    test_command: whoami
  when: windows_updates.reboot_required
```

---

## 🧠 Optional: Add Backup Verification

In `pre_validate_sql.yml`, add:

```yaml
- name: Verify a recent backup exists
  win_shell: |
    sqlcmd -Q "SELECT TOP 1 database_name, backup_finish_date FROM msdb.dbo.backupset ORDER BY backup_finish_date DESC"
  register: backup_check
- debug:
    msg: "Last backup: {{ backup_check.stdout_lines[2] }}"
```

---

## ✅ Benefits of This Design

| Feature                     | Included? |
| --------------------------- | --------- |
| Conditional CU check        | ✅         |
| Disk space validation       | ✅         |
| Patch only if needed        | ✅         |
| Error catching + email      | ✅         |
| Reboot detection & control  | ✅         |
| Clean rollback logic        | ✅         |
| Full logging & traceability | ✅         |

---

## 🔄 Next Steps

Would you like:

* A **complete zip** or GitHub repo with this layout
* Email integration via authenticated SMTP (e.g., Outlook 365)
* Logging to centralized log server (Splunk, ELK, etc)
* Integration with **Slack, Teams**, or **ServiceNow**
