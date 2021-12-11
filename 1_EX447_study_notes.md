# EX447 - Summary

## 1. Configure ansible to manage nodes
Using the **control-node**, use ansible to create a wheel user **svc.ansible** that uses SSH-keys without a **sudo** password requirement. The provisioned (target) **managed-nodes** only have **a_user** account with sudo password.

### Task breakdown
1. Validate the ansible installation on the **control-node**. Install if needed.
2. Validate user **a_user**. Create ssh keypair if needed.
3. Validate that **a_user** can login and sudo on the **target1** node. Copy ssh-key to the **target1** node.
4. Create inventory file. 

## 2. Configure Ansible Tower
Install and configure **Ansible Tower** on a RHEL 8 server.
