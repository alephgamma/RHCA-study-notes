# EX447 - Summary

## 1. Configure ansible to manage nodes
Using the **control-node**, use ansible to create a wheel user **svc.ansible** that uses SSH-keys without a **sudo** password requirement. The provisioned (target) **managed-nodes** only have **a_user** account that can sudo with a password.

### Task breakdown
1. Validate the ansible installation on the **control-node**. Install if needed.
  ```
  [control-node ~]# yum install ansible -y
  ```
2. How are the **managed-nodes** reachable? IP, hostname, DNS? 
  ```
  [control-node ~]# ping node1
  ```
3. Update the default inventory file if needed: **/etc/ansible/inventory**
  ```
  node1 ansible_ssh=10.0.1.1
  node2 ansible_ssh=10.0.1.2
  node3 ansible_ssh=10.0.1.3
  ```
4. Is the **svc.ansible** user on the **control-node**? Create if needed.
  ```
  [control-node ~]# useradd -G 10 svc.ansible
  ```
5. Create a key-pair for the **svc.ansible** user.
  ```
  [control-node ~]# sudo su - svc.ansible
  [control-node ~]$ ssh-keygen
  ```
6. Create **svc.ansible** user and copy the pub-key to the **managed-nodes**.
  ```
  [control-node ~]$ sudo su -
  [control-node ~]# ansible -u a_user -i node1, -m user -a "name=svc.ansible group=wheel" -b
  [control-node ~]# sudo su - svc.ansible
  [control-node ~]$ ssh-copy-id node1 
  ```
7. Update **/etc/ansible/ansible.cfg**
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
8. Copy the default inventory file to **/etc/svc.ansible/inventory**
  ```
  [control-node ~]$ cp /etc/ansible/inventory /etc/svc.ansible/inventory
  ```
9. Update the sudoers file on the **managed-nodes**
  ```
  [control-node ~]$ ansible all -m lineinfile -a "dest=/etc/sudoers line='svc.ansible ALL=(ALL) NOPASSWD: ALL'"
  ```
10. Validate
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
Perform the following in git which clones a repo, then updates, modifies and creates files in the repo, and then adds those files to the repo.

### Task breakdown

1. Clone a git repo
  ```
  [control-node ~]$ yum install git -y
  [control-node ~]$ git clone https://github.com/git-username/git-repo/repo.git
  ```
2. Update, modify and create files in a git repository
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
3. Add those modified files back into the git repository
  ```
  [control-node ex447]$ git push origin master
  ````
  
## 3. Manage inventory variables
Use multiple inventory files. 

### Task breakdown

1. Structure host and group variables using multiple files per host or group
  ```
  [control-node ~]$ ansible-playbook play.yml -i webservers -i dbservers
  ```
  If there is a host variable with the same name **myvar** in both file, the one in dbservers takes precedence. 
  
2. Use special variables to override the host, port, or remote user Ansible uses for a specific host
 - **/etc/ansible/ansible.cfg**
  ```
  inventory=/home/ansible/inventory
  remote_user = centos 
  remote_port = 20022
  ```
 - **/home/ansible/inventory**
  ```
  alias1 ansible_host=10.2.3.1 ansible_ssh_port=2122 ansible_user=bob
  alias2 ansible_host=10.2.3.2 ansible_ssh_port=2122 ansible_user=sandy
  ```
3. Set up directories containing multiple host variable files for some of your managed hosts
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
4. Override the name used in the **inventory** file with a different name or IP address
  ```
  
  ```
  
## 4. Manage task execution
Control privilege execution

### Task breakdown
1. Control privilege execution

  **become** at the task level?

Run selected tasks

### Task breakdown
1. Use tags to specify tasks
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
Populate variables with data from external sources using lookup plugins

### Task breakdown
1. Use a lookup file plugin to read a file

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
2. Use lookup and query functions to template data from external sources into playbooks and deployed template files
3. Implement loops using structures other than simple lists using lookup plugins and filters
4. Inspect, validate, and manipulate variables containing networking information with filters

## 6. Delegate tasks

### Task breakdown
1. Run a task for a managed host on a different host, then control whether facts gathered by that task are delegated to the managed host or the other host
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
## 7. Install and Configure Ansible Tower

### Task breakdown
1. Get the Ansible Tower bundle: **ansible-automation-platform-setup-bundle-1.2.1-1.tar.gz**

2. Untar the bundle and cd into a directory. Example: **/home/cloud_user/**
```
# tar -xvf ansible-automation-platform-setup-bundle-1.2.1-1.tar.gz
# cd ansible-automation-platform-setup-bundle-1.2.1-1
```
3. Edit the ansible Tower inventory file to set the **admin_password** and the **pg_password**
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

4. Ensure the following rpm is installed
```
$ rpm -q python3-libselinux
python3-libselinux-2.9-3.el8.x86_64
```

5. Ensure python3 is used
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

6. Install Ansible Tower - This can take up to 20 minutes...
```
# cd ansible-automation-platform-setup-bundle-1.2.1-1
# ./setup.sh
```

7. Ensure Ansible Tower is started
```
# systemctl status ansible-tower
```

## 8. Manage access for Ansible Tower

### Task breakdown
1. Create Ansible Tower users and teams and make associations of one to the other

## 9. Manage Ansible Tower inventories and credentials

### Task breakdown
1. Manage advanced inventories
2. Create a dynamic inventory from an identity management server or a database server
3. Create machine credentials to access inventory hosts
4. Create a source control credential

## 10. Manage Ansible Tower projects
Create a project and then a job template

### Task breakdown
1. Create a project
   - Name: **MyProject**
   - SCM Type: **git**
   - SCM URL: https://github.com/git-username/git-repo/repo.git
   - SCM CREDENTIAL: Must be created as a Credential of type Source Control
   - SCM UPDATE OPTIONS
     [x] Update Revision on Launch

2. Create a job template
   - New Job Template
   - NAME: **MyJobTemplate**
   - INVENTORY: playground
   - PROJECT: **MyProject**
   - PLAYBOOK: Available from the git repo
   - CREDENTIALS:
     - cloud_user/superscret
   - OPTIONS
     - ENABLE PRIVILEGE ESCALATION

## 11. Manage Ansible Tower job workflows

### Task breakdown
1. Create a job workflow template

## 12. Work with the Ansible Tower API
Write an API scriptlet to launch a job

### Task breakdown
1. Install jq. It may be useful.
```
$ sudo yum install jq -y
```

2. Get the job id. Using the browser is best, **https://$tower-ip/api/v2/job_templates/** 
   - Look for: 
  ```
  ...
  "id": 10,
  "type": "job_template"
  ...
  ```
3. The scriptlet
  ```  
  $ echo 'curl -k -L -H "Content-Type: application/json" -X POST --user username:password https://$tower-ip/api/v2/job_templates/10/launch' > api-scriptlet.sh
  chmod u+x api-scriptlet.sh
  ```
  
## 13. Back up Ansible Tower
Back up an instance of Ansible Tower

### Task breakdown
1. Run the command in the ansible installation directory, example: **ansible-automation-platform-setup-bundle-1.2.1-1**
  ```
  $ ./setup.sh -b 
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
lrwxrwxrwx.  1 root  root     105 Dec 17 00:58 tower-backup-latest.tar.gz -> /home/cloud_user/ansible-automation-platform-setup-bundle-1.2.1-1/tower-backup-2021-12-17-00:57:56.tar.gz

```
### Bonus
1. Restore from a backup
```
$ ./setup.sh -r -e 'restore_backup_file=/path/to/tar-gz-file'
```
