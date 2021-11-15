# EX447 - Summary

## 1. Configure ansible to manage nodes
A recently provisioned system **target1** only has a **svc_admin** account and a password has been provided. Using a **control-node**, use ansible to create a wheel user **svc_admin** that uses SSH-keys without a sudo password requirement

### Task breakdown
1. Validate ansible installation on the **control-node**. Install if needed.
2. Validate user **svc_admin**. Create ssh keypar if needed.
3. Validate that **svc_admin** can login and sudo on the **target1** node. Copy ssh-key to the **target1** node.
4. Create inventory file. 
