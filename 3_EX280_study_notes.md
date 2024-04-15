# EX280 - Summary

## Prerequisites
* Set the text editor
```
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
```
* The missing manual: `man limitrange`
```
oc explain limitrange --recursive
oc explain limitrange.spec.limits
```
## 1. Install the OpenShift CodeReady Container (crc) Platform (Template)

### Task
Install `crc` on RHEL 8. 

### Requirements (Optional)
* New settings

### Task breakdown
1.1. Do this
```
this-command
```
1.2. And then that
```
that-command
```

## 2. Identity Providers

### Task
Configure `htpasswd` as the Identity Provider

### Requirements
* Users
  * `manager` / `manager123`
  * `qauser` / `qauser123`
  * `devuser` / `devuser123`

### Task breakdown
2.1. Install `httpd-tools`
```
sudo yum install httpd-tools -y
```
2.2. Create the `htpasswd` file and add the users
```
sudo htpasswd -c -B -b /tmp/users.htpasswd manager manager123
sudo htpasswd -b /tmp/users.htpasswd qauser qauser123
sudo htpasswd -b /tmp/users.htpasswd devuser devuser123
```
2.3. Create the secret `localusers` in the NAMESPACE `openshift-config`
```
oc create secret generic localusers \
--from-file htpasswd=/tmp/users.htpasswd \
-n openshift-config
```
2.4. Get the oauth cluster RESOURCE, but first make a back up 
```
oc get oauth cluster -o yaml > oauth-original.yml
cp oauth-original.yml oauth.yml
```
2.5. Or edit the file in place
```
oc edit oauth.config.openshift.io/cluster
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
oc replace -f oauth.yml
```

## 3. Role-Based Access and Groups

### Task
Create projects, groups and manage users with the respective roles

### Requirements
* Grant the user `manager` the cluster-role `cluster-admin`
* Create projects: `wonderland` `zland`
* Create groups: `admin-group` `dev-group` `qa-group`
* Grant the `self-provisioner` role ONLY to the group `dev-group`
* Add users to groups and roles
  * Add the user `manager` to the `admin-group`
  * Add the user `dev-user` to the `dev-group`
  * Grant the role `admin` to `manager` to projects `wonderland` `zland`
  * Grant the role `edit` to `devuser` to the project `wonderland`
  * Grant the role `view` to `qauser` to the project `zland`
* Remove the user `kubadmin`

### Task breakdown
3.1. Grant the role `cluster-admin` the user `manager`
```
oc adm policy add-cluster-role-to-user cluster-admin manager
```
3.2. Create projects: `wonderland` `zland`
```
oc new-project wonderland
oc new-project zland
```
```
for P in wonderland zland ; do oc new-project ${P} ; done
```
3.3. Create the groups: `admin-group` `dev-group` `qa-group`
```
oc adm groups new admin-group
oc adm groups new dev-group
oc adm groups new qa-group
```
```
for G in wonderland zland ; do oc adm groups new ${G} ; done
```
3.4. Remove the ability for ALL users to create new projects
```
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```
3.5. Grant the role `self-provisioner` to the `dev-group`
```
oc adm policy add-cluster-role-to-group self-provisioner dev-group
```
3.6. Add `manager` to the group `admin-group`
```
oc adm groups add-users admin-group manager 
```
3.7. Grant `manager` the role `admin` in the projects `wonderland` `zland`
```
oc adm policy add-role-to-user admin manager -n wonderland
oc adm policy add-role-to-user admin manager -n zland
```
3.8. Add `devuser` the group `dev-group`
```
oc adm policy add-role-to-user view devuser -n wonderland 
```
3.9. Grant `devuser` the role `edit` to the project `wonderland`
```
oc adm policy add-role-to-user view devuser -n wonderland 
```
3.10. Grant `qauser` the role `view` to the project `zland`
```
oc adm groups add-users admin-group manager -n zland
```
3.11. Remove the `kubeadmin` user from the cluster
```
oc delete secrets kubeadmin -n kube-system
```
**NOTE**: Do not delete `kubeadmin` from CRC

## 4. Security Context Constraints 

### Task
Configure Security Context Constraints (SCC)

### Requirements
* Using project `gitlab-project` deploy a gitlab server from `quay.io/redhattraining/gitlab-ce:8.4.3-ce.0`

### Task breakdown
4.1. Create the gitlab project and deploy the app
```
oc new-project gitlab-project
oc new-app --image quay.io/redhattraining/gitlab-ce:8.4.3-ce.0
```
4.2. Create a serviceaccount `application-sa`
```
oc create serviceaccount application-sa -n gitlab-project
```
4.3. As `kubeadmin` assign the SCC `anyuid` to the Service Account `application-sa`
```
oc login -u kubeadmin -p SUPER-SECRET https://api.crc.testing:6443
oc adm policy add-scc-to-user anyuid -z application-sa
oc login -u developer -p developer https://api.crc.testing:6443
```
4.4. Assign the `application-sa` Service Account to the gitlab deployment
```
oc set serviceaccount deployment.apps/gitlab-ce application-sa -n gitlab-project
```

## 5. Secure routes: `edge`
Create a secure `edge` route to the pod

### Requirements
* Create a cert and key
* Using a new project deploy an http server with TLS
* Deploy from `quay.io/redhattraining/hello-world-secure:v1.0`
* Ingress route from the FQDN

### Task breakdown
5.1. Create the cert and key
```
openssl req -x509 -newkey rsa:4096 -nodes -sha256 -out edge.crt -keyout edge.key -days 3650 \
-subj '/C=US/ST=Indiana/L=Hawkins/O=DOE/OU=National Labs/CN=hawkins.doe.gov/emailAddress=nunya@bidness.com'
```
5.2. Create the secure http server project and deploy the app
```
oc new-project hello-secure-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
5.3. Create the `edge` route
```
APPS=`oc whoami --show-console | cut -d'.' -f2,3`
oc create route edge \
--hostname hello.$APPS \
--service hello-world-nginx \
--key edge.key \
--cert edge.crt
```
5.4. Verify
```
curl -k hello.apps-crc.testing
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

## 6. Secure routes: `passthrough`

### Task
Create a secure `passthrough` route to the pod

### Requirements
* Create a cert and key
* Using a new project deploy an http server with TLS
* Deploy from `quay.io/redhattraining/hello-world-secure:v1.0`
* Set a local volume for the cert and key

### Task breakdown
6.1. Create the cert and key
```
openssl req -x509 -newkey rsa:4096 -nodes -sha256 -out passthrough.crt -keyout passthrough.key -days 3650 \
-subj '/C=US/ST=Indiana/L=Hawkins/O=DOE/OU=National Labs/CN=hawkins.doe.gov/emailAddress=nunya@bidness.com'
```
6.2. Create the secure http server project and deploy the app
```
oc new-project hello-secure-project
oc new-app --name hello-secure --image quay.io/redhattraining/hello-world-secure:v1.0
```
6.3. Create the TLS `passthrough` secret
```
oc create secret tls passthrough \
--key passthrough.key \
--cert passthrough.crt
```
6.4. Create the volume that will have the cert and key
```
oc set volumes deployment.apps/hello-secure \
--add \
--type secret \
--secret-name passthrough \
--mount-path /run/secrets/nginx
```
6.5. Create secure edge route
```
oc create route passthrough --service hello-secure 
```
6.6. Verify
```
curl -k hello.apps-crc.testing
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```
## 7. Secret literals

### Task
Create a secret from **key: value** pair(s) and apply to a deployment

### Requirements
* `root_password: rootpass`
* `user: mysqluser`
* `password: mysqlpass`
* `database: mysqldb`

### Task breakdown
7.1. Create the project and deploy the application
```
oc new-project mysql-project
oc new-app mysql \
-n mysql \
--name=mysql-name \
-l app=mysql-label
```
7.2. Create the secret from **key: value** pairs
```
oc create secret generic mysql-secret \
--from-literal root_password=rootpass \
--from-literal user=mysqluser \
--from-literal password=mysqlpass \
--from-literal database=mysqldb
```
7.3. Set the deployment RESOURCE to use environment variables
```
oc set env deployment.apps/mysql-name --from secret/mysql-secret --prefix MYSQL_
```

## 8. Labeling nodes

### Task
Label a node with a tag **ENV** and set it to values **( key: value )**

### Requirements
* `master01: prod`
* `master02: test`
* `master03: dev`

### Task breakdown
8.1. As `kubeadmin` (or a user with the `cluster-admin` role) get the nodes
```
oc get nodes --show-labels
NAME       STATUS   ROLES           AGE    VERSION           LABELS
master01   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master01,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
master02   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master02,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
master03   Ready    master,worker   621d   v1.23.3+e419edf   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master03,kubernetes.io/os=linux,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
```
8.2. Set the node tag **( k=v )**
```
oc label node master01 env=prod
oc label node master02 env=test
oc label node master03 env=dev
```
8.3. View the label `env`
```
oc get node -L env
NAME       STATUS   ROLES           AGE    VERSION           ENV
master01   Ready    master,worker   621d   v1.23.3+e419edf   prod
master02   Ready    master,worker   621d   v1.23.3+e419edf   test
master03   Ready    master,worker   621d   v1.23.3+e419edf   dev
```
8.4. Create a project and application 
```
oc new-project pods-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0 --name hello -n pods-project
```
8.5. Get the node on which the pod is running 
```
oc get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
hello-787445fd88-tcqv9   1/1     Running   0          67s   10.9.0.41   master01   <none>           <none>
```
8.6. Edit a deployment to use a tagged node
```
oc edit deployment/hello
...output omitted...
spec:
  template:
    spec:
      dnsPolicy: ClusterFirst
      nodeSelector:
        env: dev
...output omitted...
```
8.7. Get the node on which the pod is running 
```
oc get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
hello-b64bdf567-t5r4v   1/1     Running   0          65s   10.10.0.72   master03   <none>           <none>
```
## 9. ResourceQuotas

### Task
Create a ResourceQuota

### Requirements
* pod count: 3
* cpu allocated: 200 millicores
* memory: 2 GB

### Task breakdown
9.1. Create the project and the app
```
oc new-project zland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
9.1. Two ways to do this

9.1.1. Create and apply the `resourcequota` YAML CRD
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-resource
  namespace: zland-project
spec:
  hard:
    pods: "3"
    memory: "2Gi"
    cpu: "200m"
    replicationcontrollers: ""
    services: ""
```
9.1.2. Or create the Resource at the CLI
```
oc create quota quota-resource --hard pods=3,memory=2Gi,cpu=200m -n zland-project
```
## 10. LimitRanges

### Task
Create a LimitRange

### Requirements
* Set the Pods and Containers to:
  * max cpus: 2
  * min cpus: 200 millicores
  * max memory: 1 GB
  * min memory: 16 MB
* Set DefaultRequest for Pods to:
  * cpu: 100 millicores

### Task breakdown
10.1. Create the project and the app
```
oc new-project wonderland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
10.1. Create and apply the `limitrange` YAML CRD
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
      defaultRequest:
        cpu: "100m"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "16Mi"
```
10.2. Apply the limits YAML file
```
oc create -f limits.yaml -n wonderland
```
## 11. Scaling

### Task 1
Manually scale replicas

### Requirements
* Set the replicas to 2

### Task 1 breakdown
11.1. Create the project and the app
```
oc new-project zland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
11.2. Get the deployment RESOURCE
```
oc scale --replicas 2 deploymentconfig.apps.openshift.io/postgresql
```
11.3. Increase the amount of replicas
```
oc scale --replicas 2 deploymentconfig.apps.openshift.io/postgresql
```
### Task 2
Horizontal Pod Autoscaling `hpa`

### Requirements
* min: 1
* max: 3
* cpu percentage: 75

### Task 2 breakdown
11.3. Dynamically scale
```
oc autoscale deployment.app/postgresql --min 1 --max 3 --cpu-percent 75
```
