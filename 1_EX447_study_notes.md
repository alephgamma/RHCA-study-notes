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

### Task breakdown

1. Clone a git repo
  ```
  [control-node ~]$ yum install git -y
  [control-node ~]$ git clone https://github.com/git-username/git-repo/repo.git
  ```
2. Update, modify and create files in a Git repository
  ```
  [control-node ~]$ cd git-repo
  [control-node git-repo]$ vi README.md
  [control-node git-repo]$ git add README.md
  [control-node git-repo]$ git commit -m "Added README.md"
  ```
3. Add those modified files back into the Git repository
  ```
  [control-node ex447]$ git push origin master
  ````
  
## 3. Manage inventory variables

### Task breakdown

1. Structure host and group variables using multiple files per host or group
  ```
  [control-node ~]$ ansible-playbook play.yml -i webservers -i dbservers
  ```
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
4. Override the name used in the inventory file with a different name or IP address

## 4. Manage task execution

### Task breakdown
1. Control privilege execution
2. Run selected tasks

## 5. Transform data with filters and plugins

Delegate tasks
1. Populate variables with data from external sources using lookup plugins
2. Use lookup and query functions to template data from external sources into playbooks and deployed template files
3. Implement loops using structures other than simple lists using lookup plugins and filters
4. Inspect, validate, and manipulate variables containing networking information with filters

## 6. Delegate tasks

### Task breakdown
1. Run a task for a managed host on a different host, then control whether facts gathered by that task are delegated to the managed host or the other host

## 7. Configure Ansible Tower

### Task breakdown
1. Install and configure **Ansible Tower** on a RHEL 8 server.
