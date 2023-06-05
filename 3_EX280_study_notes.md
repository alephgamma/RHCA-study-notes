# EX280 - Summary

## 1. Install the OpenShift Container Platform

### Task
Install the CodeReady Containers (or OpenShift 4.6) on RHEL 8. 

### Requirements
* Create a user with the cluster-admin role

### Task breakdown
1.1. Do this
```
$ this-command
```
1.2. And then that
```
$ that-command
```

## 2. Configure an Identity Provider

### Task
Configure htpasswd as the Identity Provider

### Task breakdown
2.1. Install httpd-tools
```
$ sudo yum install httpd-tools -y
```
2.2. Create the htpasswd file and add the users: admin redhat
```
$ sudo htpasswd -c -B -b /etc/users.htpasswd admin admin123
$ sudo htpasswd -b /etc/users.htpasswd redhat redhat123
```
2.3. Create the secret localusers in the NAMESPACE: openshift-config
```
$ oc create secret generic localusers \
--from-file htpasswd=/etc/users.htpasswd \
-n openshift-config
```
2.4. Replace the file: oauth.yml
```
$ oc replace -f oauth.yml
```

## 3. Configure Role-Based Access and Groups

### Task
Create roles, groups and manage users

### Task breakdown
3.1. Grant the role cluster-admin the the user admin
```
$ oc adm policy add-cluster-role-to-user cluster-admin admin
Warning: User 'admin' not found
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"
```
3.2. Add the groups:  dev-group  qa-group
```
$ oc adm groups new dev-group

$ oc adm groups new qa-group
```
3.3. Add the user admin to the group: dev-group
```
$ oc adm groups add-users dev-group admin
```
3.4. Remove the ability for ALL users to create new projects
```
$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```
3.5. Grant the role self-provisioner to the dev-group
```
$ oc adm policy add-cluster-role-to-group self-provisioner dev-group
```
3.6 Remove the kubeadmin user from the cluster
```
$ oc delete secrets kubeadmin -n kube-system
```

## 4. Configure Role-Based Access and Groups

### Task
Configure Security Context Constraints

### Task breakdown
