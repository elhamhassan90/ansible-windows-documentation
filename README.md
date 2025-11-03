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

### 3. 🔐 Handling Vault Password: Create, Use, and Secure (Easy Steps)

This project uses **Ansible Vault** to protect sensitive data and credentials.  
Below are simple, practical steps to create a vault password file, configure `ansible.cfg` to use it, and run playbooks safely.

---

### 1️⃣ Create a Vault Password File (Local & Private)

Create a small file that contains only the vault password.  
Do this on your **control machine**.

```bash
# Create the file and add your password (replace <your-password> with your actual password)
echo '<your-password>' > /path/to/ansible_vault_pass.txt

# Restrict file permissions so only you can read it
chmod 600 /path/to/ansible_vault_pass.txt
```
### 2️⃣ Configure Ansible to Use the Vault Password File

Open your project’s ansible.cfg file and add (or uncomment) this line under [defaults]:
```
[defaults]
vault_password_file = /path/to/ansible_vault_pass.txt
```
### 3️⃣ Create or Edit Vaulted Files
```
# Create a new vaulted file interactively
ansible-vault create group_vars/all/vault.yml

# Edit an existing vaulted file
ansible-vault edit group_vars/all/vault.yml
```
### 4️⃣ Run Playbooks with Vault
```
# Run playbook and ask for vault password interactively
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass

# Run playbook using the password file (recommended, but NEVER commit it)
ansible-playbook -i inventory.ini playbook.yml --vault-password-file /path/to/ansible_vault_pass.txt
```


### 4. Configure Inventory File
Created an **Ansible inventory file**  that defines Windows hosts and connection settings.
Sensitive credentials (like ansible_password) are securely stored in **Ansible Vault** and referenced here as variables.

```
[all:vars]
ansible_user=domainname\ansible
ansible_password="{{ ansible_password }}"
ansible_connection=winrm
ansible_winrm_port=5985
ansible_winrm_server_cert_validation=ignore
ansible_winrm_transport=ntlm

[servers-group1]
servername1 ansible_host=<IP-address>

[servers-group2]
servername2 ansible_host=<IP-address>
```

### 3. Create Playbook to Collect Time Information

Developed an Ansible Playbook that retrieves:

    Current system time

    Timezone configuration

    Windows Time Service (W32Time) status

```
---
- name: Collect time, timezone, and W32Time info from selected Windows groups (unique hosts)
  hosts: "servers-group1,servers-group2"
  gather_facts: no
  ignore_unreachable: yes
  vars:
    output_dir: "/home/ans/windows-elham/results"
  vars_prompt:
    - name: ansible_vault_password
      prompt: "Vault Password"
      private: yes

  tasks:
    - name: Ensure output directory exists
      file:
        path: "{{ output_dir }}"
        state: directory
      delegate_to: localhost

    - block:
        - name: Run PowerShell command on reachable hosts
          ansible.windows.win_shell: |
            $date = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
            $tz = tzutil /g
            $service = Get-Service W32Time | Select-Object Status, StartType
            Write-Output "$env:COMPUTERNAME,$date,$tz,$($service.Status),$($service.StartType),Reachable"
          register: time_info

        - name: Save reachable host result
          copy:
            content: "{{ time_info.stdout }}"
            dest: "{{ output_dir }}/{{ inventory_hostname }}.csv"
          delegate_to: localhost

      rescue:
        - name: Save unreachable host result
          copy:
            content: "{{ inventory_hostname }},N/A,N/A,N/A,N/A,Unreachable"
            dest: "{{ output_dir }}/{{ inventory_hostname }}.csv"
          delegate_to: localhost


- name: Combine all results into one CSV file
  hosts: localhost
  gather_facts: no
  vars:
    output_dir: "/home/ans/windows-elham/results"

  tasks:
    - name: Create CSV header
      copy:
        content: "Server,Date,Timezone,W32Time_Status,StartType,ConnectionStatus\n"
        dest: "{{ output_dir }}/time_check_results.csv"

    - name: Append data from each host file (unique servers only)
      shell: |
        cat {{ output_dir }}/*.csv | grep -v "Server,Date" | sort -t',' -k1,1 -u >> {{ output_dir }}/time_check_results.csv

    - name: Clean up temporary host CSV files
      file:
        path: "{{ item }}"
        state: absent
      with_fileglob: "{{ output_dir }}/*.csv"
      when: item != "{{ output_dir }}/time_check_results.csv"

    - name: Show total unique servers
      shell: awk -F',' 'NR>1{print $1}' {{ output_dir }}/time_check_results.csv | sort -u | wc -l
      register: total_servers

    - name: Show final result
      debug:
        msg:
          - "✅ Final CSV saved at {{ output_dir }}/time_check_results.csv"
          - "🧮 Total unique servers: {{ total_servers.stdout }}"
```


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
