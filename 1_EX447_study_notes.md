# EX447 - Summary

## 1. Configure ansible to manage nodes
Using the **control-node**, use ansible to create a wheel user **svc.ansible** that uses SSH-keys without a **sudo** password requirement. The provisioned (target) **managed-nodes** only have **a_user** account that can sudo with a password.

### Task breakdown
1. How are the **managed-nodes** reachable? IP, hostname, DNS? 
  ```
  ping node1
  ```
2. Update the inventory file if needed: **/etc/ansible/inventory**
  ```
  node1 ansible_ssh=10.0.1.1
  ```
3. Validate the ansible installation on the **control-node**. Install if needed.
4. Is the **svc.ansible** user on the **control-node**? Create if needed.
5. Create a key-pair for the **svc.ansible** user.
6. Create **svc.ansible** user and copy the pub-key to the **managed-nodes**.
7. Update the **/etc/ansible/ansible.cfg** file:
  ```
  [defaults]
  inventory = /etc/ansible/inventory
  remote_user = ansible
  host_key_checking = False
  private_key_file = /home/svc.ansible/.ssh/id_rsa

  [privilege_escalation]
  become = true
  become_user = root
  become_method = sudo
  become_ask_pass = false
  ```
7. Update the inventory file? 

## 2. Configure Ansible Tower
Install and configure **Ansible Tower** on a RHEL 8 server.
