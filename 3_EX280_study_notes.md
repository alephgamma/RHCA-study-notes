# EX280 - Summary

## 1. Install the OpenShift CodeReady Container (crc) Platform (Template)

### Task
Install `crc` on RHEL 8. 

### Requirements (Optional)
* New settings

### Task breakdown
1.1. Do this
```
$ this-command
```
1.2. And then that
```
$ that-command
```

## 2. Identity Providers

### Task
Configure `htpasswd` as the Identity Provider

### Task breakdown
2.1. Install `httpd-tools`
```
$ sudo yum install httpd-tools -y
```
2.2. Create the `htpasswd` file and add the users
* `manager` / `manager123`
* `redhat` / `redhat123`
```
$ sudo htpasswd -c -B -b /etc/users.htpasswd manager manager123
$ sudo htpasswd -b /etc/users.htpasswd redhat redhat123
```
2.3. Create the secret `localusers` in the NAMESPACE `openshift-config`
```
$ oc create secret generic localusers \
--from-file htpasswd=/etc/users.htpasswd \
-n openshift-config
```
2.4. Get the oauth cluster RESOURCE, but first make a back up 
```
$ oc get oauth cluster -o yaml > oauth-original.yml
$ cp oauth-original.yml oauth.yml
```
2.5. Or edit the file in place
```
$ oc edit oauth.config.openshift.io/cluster
```
2.6. Edit the oauth file 
``` 
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: localusers
    mappingMethod: claim
    name: htpasswd-file
    type: HTPasswd
```
2.7. Replace the file `oauth.yml`
```
$ oc replace -f oauth.yml
```

## 3. Role-Based Access and Groups

### Task
Create roles, groups and manage users

### Task breakdown
3.1. Grant the role cluster-admin the the user `manager`
```
$ oc adm policy add-cluster-role-to-user cluster-admin manager
```
3.2. Add the groups `admin-group` `dev-group` `qa-group`
```
$ oc adm groups new admin-group
$ oc adm groups new dev-group
$ oc adm groups new qa-group
```
3.3. Add the user `manager` to the group `admin-group`
```
$ oc adm groups add-users admin-group manager
```
3.4. Remove the ability for ALL users to create new projects
```
$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```
3.5. Grant the role `self-provisioner` to the `dev-group`
```
$ oc adm policy add-cluster-role-to-group self-provisioner dev-group
```
3.6. Remove the `kubeadmin` user from the cluster
```
$ oc delete secrets kubeadmin -n kube-system
```
NOTE: Do not delete for CRC

## 4. Security Context Constraints 

### Task
Configure Security Context Constraints (SCC)

### Requirements
* Using a new project deploy gitlab server from `quay.io/redhattraining/gitlab-ce:8.4.3-ce.0`

### Task breakdown
4.1. Create the gitlab project and deploy the app
```
$ oc new-project gitlab-project
$ oc new-app --image quay.io/redhattraining/gitlab-ce:8.4.3-ce.0
```
4.2. Create a serviceaccount `application-sa`
```
$ oc create serviceaccount application-sa
```
4.3. As `kubeadmin` assign the SCC `anyuid` to the Service Account `application-sa`
```
$ oc login -u kubeadmin -p SUPER-SECRET https://api.crc.testing:6443
$ oc adm policy add-scc-to-user anyuid -z application-sa
$ oc login -u developer -p developer https://api.crc.testing:6443
```
4.4. Assign the `application-sa` Service Account to the gitlab deployment
```
$ oc set serviceaccount deployment.apps/gitlab-ce application-sa
```

## 5. Secure routes

### Task
Create a secure passthrough route to the pod

### Requirements
* Create a cert and key
* Using a new project deploy an http server with TLS
* Deploy from quay.io/redhattraining/hello-world-secure:v1.0

### Task breakdown
5.1. Create the cert and key
```
$ openssl req -x509 -newkey rsa:4096 -nodes -sha256 -out passthrough.crt -keyout passthrough.key -days 3650 \
-subj '/C=US/ST=Indiana/L=Hawkins/O=DOE/OU=National Labs/CN=hawkins.doe.gov/emailAddress=nunya@bidness.com'
```
5.2. Create the secure http server project and deploy the app
```
$ oc new-project hello-secure-project
$ oc new-app --name hello-secure --image quay.io/redhattraining/hello-world-secure:v1.0
```
5.3. Create the TLS secret
```
$ oc create secret tls passthrough \
--key passthrough.key \
--cert passthrough.crt
```
5.4. Create the volume that will have the cert and key
```
$ oc set volumes deployment.apps/hello-secure \
--add \
--type secret \
--secret-name passthrough \
--mount-path /run/secrets/nginx
```
5.5. Create secure edge route
```
$ oc create route passthrough --service hello-secure 
```

## 6. Secret literals

### Task
Create a secret from **key:value** pair(s) and apply to a deployment

### Task breakdown
6.1. Create the project and deploy the application
```
$ oc new-project mysql-project
$ oc new-app mysql \
-n mysql \
--name=mysql-name \
-l app=mysql-label
```
6.2. Create the secret from **key:value** pairs
* root_password=rootpass
* user=mysqluser
* password=mysqlpass
* database=mysqldb
```
$ oc create secret generic mysql-secret \
--from-literal root_password=rootpass \
--from-literal user=mysqluser \
--from-literal password=mysqlpass \
--from-literal database=mysqldb
```
6.3. Set the deployment RESOURCE to use environment variables
```
$ oc set env deployment.apps/mysql-name --from secret/mysql-secret --prefix MYSQL_
```

## 7. Labeling nodes

### Task
Label a node with a tag **ENV** and set it to PROD **( k=v )**

### Task breakdown
7.1. As `kubeadmin` (or a user with the `clusteradmin` role) get the nodes
```
$ oc get nodes --show-labels
NAME       STATUS   ROLES           AGE    VERSION           LABELS
master01   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master01,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
master02   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master02,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
master03   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master03,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
```
7.2. Set the node tag **( k=v )**
```
$ oc label node master01 env=prod
```
7.3. View the label `env`
```
$ oc get node -L env
NAME       STATUS   ROLES           AGE    VERSION           ENV
master01   Ready    master,worker   621d   v1.23.3+e419edf   prod
master02   Ready    master,worker   621d   v1.23.3+e419edf
master03   Ready    master,worker   621d   v1.23.3+e419edf
```
7.3. Create a project and application 
```
$ oc new-project pods-project
$ oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0 --name hello -n pods-project
```
7.3. Get the node on which the pod is running 
```
$ oc get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
hello-787445fd88-tcqv9   1/1     Running   0          67s   10.9.0.41   master01   <none>           <none>
```
7.4. Edit a deployment to use a tagged node
```
$ oc get deployment
```
## 8. ResourceQuotas

### Task
Create a ResourceQuota

### Task breakdown
```
$ oc create quota quota-resource --hard pods=3,memory=2Gi,cpu=200m -n NAMESPACE
```
## 9. LimitRanges

### Task
Create a LimitRange

### Task breakdown
9.1. Create limits yaml file
```
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "16Mi"

```
9.2. Apply the limits file
```
$ oc create -f limits.yaml -n NAMESPACE
```
## 10. Scaling

### Task 1
Manually scale replicas to 2

### Task breakdown
10.1. Increase the amount of replicas
```
$ oc scale --replicas 2 deploymentconfig.apps.openshift.io/postgresql
```
### Task 2
Horizontal Pod Autoscaling (hpa)

### Task breakdown
10.2. Dynamically scale
```
$ oc autoscale deployment.app/postgresql --min 1 --max 3 --cpu-percent 75
```
