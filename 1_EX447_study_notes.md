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
7. Update the **/etc/ansible/ansible.cfg** file:
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
9. Update the sudoers file on the **managed-nodes**.
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
## 2. Configure Ansible Tower
Install and configure **Ansible Tower** on a RHEL 8 server.
