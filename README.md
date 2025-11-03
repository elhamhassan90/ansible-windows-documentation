# ansible-windows-documentation
ansible-windows-documentation
# Windows Servers Automation using Ansible

## 📘 Overview
This documentation describes the steps I implemented at work to automate Windows Server management using **Ansible**.  
The goal was to simplify server administration tasks such as collecting time, timezone, and Windows Time Service (W32Time) status from multiple servers simultaneously.

---

## 🧩 Environment Setup
- **Control Node:** CentOS / RHEL Server with Ansible installed  
- **Managed Nodes:** Windows Servers (Domain joined)  
- **Connection Method for WindowsServers:** WinRM (Ansible connecting to Windows hosts) by activating WinRM on windowserver
- **Connection Method for Ansible:** pywinrm package installed on linux so ansible can control winrm
---

## ⚙️ Configuration Steps

### 1. Enable WinRM on Windows Servers
Configure WinRM service to allow Ansible connections securely:

open powershell as administrator 

```
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Service\Auth\Basic -Value $true
Set-Item WSMan:\localhost\Service\AllowUnencrypted -Value $true
Set-Item -Path WSMan:\localhost\Client\AllowUnencrypted -Value $true
Restart-Service WinRM
Set-NetFirewallRule -Name "WINRM-HTTP-In-TCP" -Enabled True
```

### 2. Configure Ansible/Linux Server 

```
#ansible install
sudo dnf install epel-release -y  
sudo dnf install ansible -y
ansible --version #check ansible installed
#pywinrm install (package requird for linux to deal with windows)
sudo apt install python3-pip -y
pip install "pywinrm>=0.3.0"
pip show pywinrm
```

### 3. Vault handling:
- `#vault_password_file = /home/ans/windows/group_vars/all/vault_pass.txt` — this line shows how we can point Ansible to a file that returns the vault password. In this repository the line is commented out to avoid exposing secrets.
- For secure usage, we use `ansible-vault` to encrypt sensitive variables and either pass the password at runtime (`--ask-vault-pass`) or supply a `--vault-password-file` that is excluded from version control.






### 2. Configure ansible.cfg File (Configuration file of ansible)

```
cat ansible.cfg
[defaults]
#vault_password_file = /home/ans/windows/group_vars/all/vault_pass.txt
forks = 10
inventory = ./inventory.ini
host_key_checking = False
deprecation_warnings = False
connection_timeout = 120
operation_timeout_sec = 120
[privilege_escalation]

[connection]
pipelining = True
```


### 2. Configure Inventory File

Created an Ansible inventory file with Windows server hostnames and connection variables:
```
[all:vars]
ansible_user=domainname\ansible
ansible_password="{{ ansible_password }}"
ansible_connection=winrm
ansible_winrm_port=5985
ansible_winrm_server_cert_validation=ignore
ansible_winrm_transport=ntlm

[jump_servers]
WAVZ-JUMP-SERVE ansible_host=172.27.225.4

[windows]
APPSERVER
CHILD-1
DEV-DB
DIM-WEBSERVER

[windows:vars]
ansible_user=Administrator
ansible_password=YourPassword
ansible_connection=winrm
ansible_winrm_transport=basic

### 3. Create Playbook to Collect Time Information

Developed an Ansible Playbook that retrieves:

    Current system time

    Timezone configuration

    Windows Time Service (W32Time) status

---
- name: Collect Time and Service Info from Windows Servers
  hosts: windows
  gather_facts: no
  tasks:
    - name: Get current time
      win_command: powershell Get-Date
      register: time_output

    - name: Get timezone
      win_command: powershell Get-TimeZone
      register: timezone_output

    - name: Get Windows Time Service status
      win_service_info:
        name: W32Time
      register: w32time_output

    - name: Display results
      debug:
        msg:
          - "Time: {{ time_output.stdout }}"
          - "Timezone: {{ timezone_output.stdout }}"
          - "W32Time Status: {{ w32time_output.services[0].state }}"

📄 Output Example

When the playbook runs, it shows structured results for each server:

TASK [Display results] **************************************************
ok: [APPSERVER] => {
  "msg": [
    "Time: 10/30/2025 11:20:12 AM",
    "Timezone: Egypt Standard Time",
    "W32Time Status: running"
  ]
}

🏁 Result

This automation reduced manual effort required to log into each Windows Server.
The playbook now provides a quick, centralized way to:

    Verify system time consistency across servers

    Check timezone alignment

    Ensure the Windows Time Service is running correctly

🧠 Skills Demonstrated

    Ansible for Windows automation

    YAML scripting for playbooks

    Remote management via WinRM

    Basic PowerShell integration

    Infrastructure documentation and version control using GitHub

👩‍💻 Author

Elham Hasan
LinkedIn Profile
