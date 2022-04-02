# EX447 - Summary

## 1. Configure ansible to manage nodes
### Task
Use ansible on the **control-node** to create a wheel user **svc.ansible** that uses SSH-keys without a **sudo** password requirement. The provisioned (target) **managed-nodes** only have **a_user** account that can sudo with a password.

### Task breakdown
1.1. Validate the ansible installation on the **control-node**. Install if needed.
  ```
  [control-node ~]# yum install ansible -y
  ```
1.2. How are the **managed-nodes** reachable? IP, hostname, DNS? 
  ```
  [control-node ~]# ping node1
  ```
  Usually the **/etc/hosts** file is updated with "IP hostname"
  
1.3. Update the default inventory file: **/etc/ansible/hosts** if needed. The following notes will use:  **/etc/ansible/inventory**
  ```
  [proxy]
  node1 ansible_ssh=10.0.1.1
  [webserver]
  node2 ansible_ssh=10.0.1.2
  node3 ansible_ssh=10.0.1.3
  [database]
  node4 ansible_ssh=10.0.1.4
  node5 ansible_ssh=10.0.1.5
  ```
  
  Groups are not defined yet, but usually there is a **proxy**, **webservers**, **dbservers** and maybe a **prod** group. 
  
1.4. Is the **svc.ansible** user on the **control-node**? Create if needed.
  ```
  [control-node ~]# useradd -G 10 svc.ansible
  ```
  
1.5. Create a key-pair for the **svc.ansible** user.
  ```
  [control-node ~]# sudo su - svc.ansible
  [control-node ~]$ ssh-keygen
  ```
  
1.6. Create **svc.ansible** user and copy the pub-key to the **managed-nodes**.
  ```
  [control-node ~]$ sudo su -
  [control-node ~]# ansible -u a_user -i node1, -m user -a "name=svc.ansible group=wheel" -K
  BECOME password: ***********
  [control-node ~]# sudo su - svc.ansible
  [control-node ~]$ ssh-copy-id node1 
  ```
  
1.7. Update **/etc/ansible/ansible.cfg**
  ```
  [defaults]
  inventory = /etc/svc.ansible/inventory
  remote_user = svc.ansible
  host_key_checking = False
  private_key_file = /home/svc.ansible/.ssh/id_rsa

  [privilege_escalation]
  become = true
  become_user = root
  become_method = sudo
  become_ask_pass = false
  ```
  
1.8. Copy the default inventory file to **/etc/svc.ansible/inventory**
  ```
  [control-node ~]$ cp /etc/ansible/inventory /etc/svc.ansible/inventory
  ```
  
1.9. Update the sudoers file on the **managed-nodes**
  ```
  [control-node ~]$ ansible all -m lineinfile -a "dest=/etc/sudoers line='svc.ansible ALL=(ALL) NOPASSWD: ALL'"
  ```
  
1.10. Validate
  ```
  [control-node ~]$ ansible all -a whoami
  node1 | CHANGED | rc=0 >>
  root
  node2 | CHANGED | rc=0 >>
  root 
  node3 | CHANGED | rc=0 >>
  root
  ```
  
## 2. Basic git 
### Task
Perform the following using **git** which clones a repo, then creates and modifies files in the repo, and then add those files to the upstream repo.

### Task breakdown
2.1. Install CLI git and add your information
  ```
  [control-node ~]$ yum install git -y
  ```
2.2. Configure git
  ```
  [control-node ~]$ git config --global user.email "nunya@bidnes.com"
  [control-node ~]$ git config --global user.name "git-username"
  ```
2.3. Clone a git repo
  ```
  [control-node ~]$ git clone https://github.com/git-username/git-repo/repo.git
  ```
2.4. Update, modify and create files in a git repository
  ```
  [control-node ~]$ cd git-repo
  [control-node git-repo]$ vi README.md
  [control-node git-repo]$ git add README.md
  [control-node git-repo]$ git commit -m "Added README.md"
  ```
  ```
  [control-node git-repo]$ vi play.yml
  
  [control-node git-repo]$ git add play.yml
  [control-node git-repo]$ git commit -m "Added play.yml"
  ```
2.5. Add those modified files back into the git repository with error messages from github.
  ```
  [control-node git-repo]$ git push origin master
  
(gnome-ssh-askpass:5151): Gtk-WARNING **: 03:25:27.225: cannot open display: 
error: unable to read askpass response from '/usr/libexec/openssh/gnome-ssh-askpass'
Username for 'https://github.com': git-username

(gnome-ssh-askpass:5161): Gtk-WARNING **: 03:25:35.539: cannot open display: 
error: unable to read askpass response from '/usr/libexec/openssh/gnome-ssh-askpass'
Password for 'https://git-username@github.com': your-super-secret-personal-access-token
  ```
Gitlab may not prompt for the username and PAT 
  
2.6. Cache the git credentials
```
[control-node git-repo]$ git config --global credential.helper cache
```
2.7. Show the git information
  ```
[control-node git-repo]$ git config -l
user.email=nunya@bidnes.com
user.name=git-username
credential.helper=cache
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
remote.origin.url=https://github.com/git-username/git-repo/repo.git
remote.origin.fetch=+refs/heads/*:refs/remotes/origin/*
branch.master.remote=origin
branch.master.merge=refs/heads/master
  ```
  
## 3. Manage inventory variables

### Background information 
The different kinds of inventory files is a flustercuck. There are four different kinds of inventory files. 
- **ini**
- **yml**: with vars sections
- **yml**: without vars sections - just to F your brain
- **json**: why the F not.
- Tower inventory files are their own thing and just ignore CLI defined inventory file sections, groups and relative directories like  **group_vars** or **host_vars**? F you gentle reader. 

There is a magical variable **ansible_group_priority** and there don't appear to be any really clear examples. The default value is 1
```
ansible_group_priority: 10 
```
**inv.yml**
```
all:
  children:
    a_group:
      hosts:
        web1.example.com:
          http_port: 8080
          ansible_group_priority: 10
    b_group:
      hosts:
        web2.example.com:
          http_port: 81
    ungrouped: {}
```

There are up to 22 different sources and with variable precendence from **high** to **low** the rabbit hole starts with:

- Command line (-e "user=billy_joe_jim_bob")
- role defaults
- inventory files
- ... turtles all the way down ...

Within the same source, the precendence is based on the inventory file structure: 
- Host
- Child group
- Parent group
- all group

<hr>

### Subtask
3.1. Structure host and group variables using multiple files per host or group.

### Subtask breakdown
3.1.1. Multiple inventory files for different groups at the command line. 
  ```
  [control-node ~]$ ansible-playbook play.yml -i webservers.yml -i dbservers.yml
  ```
  If there is a host variable with the same name **myvar** in both files, the one in dbservers takes precedence. 

### Subtask
3.2. Use special variables to override the host, port, or remote user Ansible uses for a specific host.

### Subtask breakdown
3.2.1. Special variables for host, port, or remote user can be set system-wide.
 - **/etc/ansible/ansible.cfg**
  ```
  inventory=/home/ansible/inventory
  remote_user = centos 
  remote_port = 20022
  ```
  
  Or they can be set in individual (or multiple) inventory files
 - **/home/ansible/inventory**
  ```
  node1 ansible_host=10.2.3.1 ansible_ssh_port=2122 ansible_user=billy
  node2 ansible_host=10.2.3.2 ansible_ssh_port=2122 ansible_user=joe
  ```
### Subtask
3.3. Set up directories containing multiple host variable files for some of your managed hosts

### Subtask breakdown
3.3.1. An example structure with separate directories
  ```
  inventory/     Base directory
    AWS.yml      Inventory plugin
    dyn_inv.py   Dynamic inventory script
    hosts.ini    Static inventory file
    group_vars/  Additional directory for group variables
      all.yml    Variables for all hosts
  ```
  ```
  [control-node ~]$ ansible-playbook test-play.yml -i inventory/dyn_inv.py -i inventory/hosts.ini
  ```
  The ansible structure for containing different variables per host is the **host_vars** directory. Where each file matches a hostname:
  ```
  ./host_vars/
          ├── node1
          └── node2
  ```
### Subtask
3.4. Override the name used in the **inventory** file with a different name or IP address

### Subtask breakdown
3.4.1. Where are the different names? Are the names defined as a host variable in an **inventory** file, **vars_files** or **vars** section?
  ```
  - hosts: localhost
    become: false
    vars:
      h: "{{ inventory_hostname }}"
      n: "newname"
    tasks:
    - name: inventory_hostname
      debug:
        msg: "The inventory_hostname: {{ inventory_hostname }}"
    - set_fact:
        inventory_hostname: "{{ n }}"
    - name: inventory_hostname
      debug:
        msg: "The inventory_hostname: {{ inventory_hostname }}"
  ```
3.4.2. Where are the different IP addresses? External IP adresses must be defined in the inventory file. Whereas an internal address is defined as an **ansible_fact**. For example, **ansible_default_ipv4.address**

## 4. Manage task execution
Control privilege execution

### Task breakdown
4.1. Control privilege execution. **become** at the task level.
Snippet
```
- name: Run a command as the apache user
  command: somecommand
  become: yes
  become_user: apache
```
Playbook
```
---
- hosts: localhost
  become: true
  gather_facts: false
  tasks:
    - name: task 1
      shell: whoami
      register: w
    - debug:
        msg: "{{ w.stdout }}"
    - name: task 2
      shell: whoami
      become: true
      become_user: bueller
      register: w
    - debug:
        msg: "{{ w.stdout }}"
```
Results
```
$ ansible-playbook privilege_execution.yml -K -u gonzalezlz
BECOME password:

PLAY [localhost] *************************************************************************************************************************************

TASK [task 1] ****************************************************************************************************************************************
changed: [localhost]

TASK [debug] *****************************************************************************************************************************************
ok: [localhost] => {
    "msg": "root"
}

TASK [task 2] ****************************************************************************************************************************************
changed: [localhost]

TASK [debug] *****************************************************************************************************************************************
ok: [localhost] => {
    "msg": "bueller"
}

PLAY RECAP *******************************************************************************************************************************************
localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Task breakdown
4.2. Use tags to specify tasks
  ```
  ---
  - name: First
    hosts: webservers
    tasks:
      - name: Install c-shell
        yum: name=csh state=latest
        tags: install-software

      - name: Install and ensure apache is at the latest version
        yum: name=httpd state=latest
        tags: install-software

      - name: Create an archive
        archive: path=/var/log/messages format=gz dest=/opt/backup/messages.tgz
        tags: backup
  ```
  ```
  [control-node ~]$ ansible-playbook -i inventory software.yml --tags install-software
  ```
## 5. Transform data with filters and plugins
### Task
Populate variables with data from external sources using lookup plugins

### Task breakdown
5.1. Use a lookup file plugin to read a file.
```
lookup('file','/path/to/file')
```
```
---
- hosts: all
  tasks:
    - name: Add a public key to a user
      authorized_key: 
        user: svc.ansible
        key: "{{ lookup('file','/home/svc.ansible/.ssh/id_rsa.pub') }}"
```
5.2. Use lookup and query functions to template data from external sources into playbooks and deployed template files.
```
vars:
  motd_value: "{{ lookup('file', '/etc/motd') }}"
tasks:
  - debug:
      msg: "motd value is {{ motd_value }}"
  - template:
      src: "issue.j2"
      dest: "/etc/issue"
```
File template: **issue.j2**
```
*******
{{ motd_value }}
*******
```

5.3. Implement loops using structures other than simple lists using lookup plugins and filters: 

File: **admin-users.yml**
```
---
admin-users:
  billy: { uid: 5001, shell: "/bin/bash", state: present, pubkey: "{{lookup('file','files/id_billy')}}" }
  joe: { uid: 5002, shell: "/bin/bash", state: present, pubkey: "{{lookup('file','files/id_joe')}}" }
  jim: { uid: 5003, shell: "/bin/bash", state: present, pubkey: "{{lookup('file','files/id_jim')}}" }
  bob: { uid: 5004, shell: "/bin/bash", state: present, pubkey: "{{lookup('file','files/id_bob')}}" }
```
The following playbook just creates the users:
```
---
- hosts: all
  become: yes
  vars_files:
  - admin-users.yml
  tasks:
  - name: Create users
    user:
      name: "{{ item.key }}"
      uid: "{{ item.value.uid }}"
      home: "/home/{{ item.key }}"
      groups: "wheel"
      generate_ssh_key: yes
      shell: "{{ item.value.shell }}"
      state: "{{ item.value.present}}"
    with_dict: "{{ admin-users }}"
```
Now create the keys:
```
  tasks:
    - authorized_key:
        user: "{{ item.key }}"
        state: present
        key: "{{ item.value.pubkey }}"
      with_dict: "{{ admin-users }}"
```
5.4. Inspect, validate, and manipulate variables containing networking information with filters

## 6. Delegate tasks

### Task breakdown
6.1. Run a task for a managed host on a different host, then control whether facts gathered by that task are delegated to the managed host or the other host
  ```
  ---
  - hosts: all
    become: true
    tasks:
    - yum:
        name: httpd
        state: present
    - systemd:
        name: httpd
        state: started
        enabled: yes
    - uri:
        url: http://{{ inventory_hostname }}
        method: GET
        timeout: 30
        status_code: 200
        return_content: yes
      delegate_to: localhost
      register: webresult
    - debug:
        var: {{ webresult }}
  ```
## 7. Ansible Tower
Install and Configure Ansible Tower

### Task breakdown
7.1. Get the Ansible Tower bundle: **ansible-automation-platform-setup-bundle-1.2.1-1.tar.gz**

7.2. Untar the bundle and cd into the directory. Example: **/home/$USERNAME/**
```
# tar -xvf ansible-automation-platform-setup-bundle-1.2.1-1.tar.gz
# cd ansible-automation-platform-setup-bundle-1.2.1-1
```
7.3. Edit the ansible Tower inventory file to set the **admin_password** and the **pg_password**
```
[tower]
localhost ansible_connection=local

[automationhub]

[database]

[all:vars]
admin_password=supersecret

pg_host=''
pg_port=''

pg_database='awx'
pg_username='awx'
pg_password=supersecret
...
```

7.4. Ensure the following rpm is installed
```
$ rpm -q python3-libselinux
python3-libselinux-2.9-3.el8.x86_64
```

7.5. Ensure python3 is used
```
$ alternatives --config python

There are 3 programs which provide 'python'.

  Selection	Command
-----------------------------------------------
*  1       	/usr/libexec/no-python
   2       	/usr/bin/python3
 + 3       	/usr/bin/python2

Enter to keep the current selection[+], or type selection number: 2
```

7.6. Install Ansible Tower - This can take up to 20 minutes...
```
# cd ansible-automation-platform-setup-bundle-1.2.1-1
# ./setup.sh
```

7.7. Ensure Ansible Tower is started
```
# systemctl status ansible-tower
```

## 8. Manage access for Ansible Tower
Create Ansible Tower users and teams and make associations of one to the other.

**sysadmin** group
```
billy
joe
```
**developer** group
```
jim
bob
```
### Task breakdown
8.1. Clicketty click the GUI

## 9. Manage Ansible Tower inventories and credentials

### Task breakdown
9.1. Manage advanced inventories
   - Clicketty click the GUI

9.2. Create a dynamic inventory from an identity management server or a database server.

The inventory file
```
[proxy]
node1 ansible_ssh=10.0.1.1
[webserver]
node2 ansible_ssh=10.0.1.2
node3 ansible_ssh=10.0.1.3
[database]
node4 ansible_ssh=10.0.1.4
node5 ansible_ssh=10.0.1.5
```
Test script:
```
$ cat dynamic.sh
#!/bin/bash 
if [ "$#" -gt 2 ]; then
   echo "Too many parameters"
   exit 1
elif [ "$#" -eq 1 ] && [ "$1" == "--list" ]; then
   /usr/bin/ansible-inventory -i /home/$USERNAME/inventory --list
elif [ "$#" -eq 2 ] && [ "$1" == "--host" ]; then
   /usr/bin/ansible-inventory -i /home/$USERNAME/inventory --host "$2"
else
   echo "Error"
   exit 1
fi
```
The result from the script:
```
$ ./dynamic.sh --list
{
    "_meta": {
        "hostvars": {
            "node1": {
                "ansible_ssh": "10.0.1.1"
            },
            "node2": {
                "ansible_ssh": "10.0.1.2"
            },
            "node3": {
                "ansible_ssh": "10.0.1.3"
            },
            "node4": {
                "ansible_ssh": "10.0.1.4"
            },
            "node5": {
                "ansible_ssh": "10.0.1.5"
            }
        }
    },
    "all": {
        "children": [
            "database",
            "proxy",
            "ungrouped",
            "webserver"
        ]
    },
    "database": {
        "hosts": [
            "node4",
            "node5"
        ]
    },
    "proxy": {
        "hosts": [
            "node1"
        ]
    },
    "webserver": {
        "hosts": [
            "node2",
            "node3"
        ]
    }
}
```
9.3. Create machine credentials to access inventory hosts
   - Clicketty click the GUI

9.4. Create a source control credential
   - Clicketty click the GUI
   
## 10. Manage Ansible Tower projects
Create a project and then a job template

### Task breakdown
10.1. Create a project
   - Name: **MyProject**
   - SCM Type: **git**
   - SCM URL: `https://github.com/git-username/git-repo/repo.git`
   - SCM CREDENTIAL: Must be created as a Credential of type Source Control
   - SCM UPDATE OPTIONS
     - Update Revision on Launch

10.1.1. Somehow, a customized **/etc/ansible/ansible.cfg** file may cause the following error message:
```
{
    "module_stdout": "",
    "module_stderr": "sudo: /etc/sudo.conf is owned by uid 65534, should be 0\nsudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set\n",
    "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
    "rc": 1,
    "_ansible_no_log": false,
    "changed": false
}
```
10.2. Create a job template
   - New Job Template
   - NAME: **MyJobTemplate**
   - INVENTORY: playground
   - PROJECT: **MyProject**
   - PLAYBOOK: Available from the git repo
   - CREDENTIALS:
     - $USERNAME/supersecret
   - OPTIONS
     - ENABLE PRIVILEGE ESCALATION

## 11. Manage Ansible Tower job workflows

### Task breakdown
11.1. Create a job workflow template
   1. Clicketty click the GUI

## 12. Work with the Ansible Tower API
Write an API scriptlet to launch a job

### Task breakdown
12.1. Install jq. It may be useful.
```
$ sudo yum install jq -y
```

12.2. Get the job id. Using the browser is best, **http://$tower-ip/api/v2/job_templates/** 
   - Look for, and match the name...
  ```
  ...
  "id": 10,
  "type": "job_template"
  ...
  lots of stuff
  ...
  "name": "MyProductionTemplate"
  ```
12.3. The scriptlet
  ```  
  $ echo 'curl -k -L -H "Content-Type: application/json" -X POST --user username:password http://$tower-ip/api/v2/job_templates/10/launch' > api-scriptlet.sh
  chmod u+x api-scriptlet.sh
  ```
  
## 13. Back up Ansible Tower
Back up an instance of Ansible Tower

### Task breakdown
13.1. Run the command in the ansible installation directory, example: **ansible-automation-platform-setup-bundle-1.2.1-1**
```
# ./setup.sh -b 
```
This creates a backup file in the format: **tar.gz** 
```
[ansible-automation-platform-setup-bundle-1.2.1-1]# ls -l
total 184
-rw-r--r--.  1 28845 28845    626 Jan 13  2021 backup.yml
drwxr-xr-x.  4 28845 28845     28 Jan 13  2021 bundle
drwxr-xr-x.  3 28845 28845     33 Jan 13  2021 collections
drwxr-xr-x.  2 28845 28845     17 Jan 13  2021 group_vars
-rw-r--r--.  1 28845 28845   8521 Jan 13  2021 install.yml
-rw-r--r--.  1 28845 28845   2924 Dec 13 19:18 inventory
drwxr-xr-x.  3 28845 28845   8192 Jan 13  2021 licenses
-rw-r--r--.  1 28845 28845   2506 Jan 13  2021 README.md
-rw-r--r--.  1 28845 28845   1335 Jan 13  2021 rekey.yml
-rw-r--r--.  1 28845 28845   3492 Jan 13  2021 restore.yml
drwxr-xr-x. 21 28845 28845   4096 Jan 13  2021 roles
-rwxr-xr-x.  1 28845 28845  10819 Jan 13  2021 setup.sh
-rw-------.  1 root  root  123551 Dec 17 00:58 tower-backup-2021-12-17-00:57:56.tar.gz
lrwxrwxrwx.  1 root  root     105 Dec 17 00:58 tower-backup-latest.tar.gz -> /home/$USERNAME/ansible-automation-platform-setup-bundle-1.2.1-1/tower-backup-2021-12-17-00:57:56.tar.gz

```
## 14. Ansible Tower restoral
14.1. Restore from a backup
```
$ ./setup.sh -r -e 'restore_backup_file=/path/to/tar-gz-file'
```

## 15. Checksum
15.1. Get a file from a webserver only if the sha256 checksum matches.
