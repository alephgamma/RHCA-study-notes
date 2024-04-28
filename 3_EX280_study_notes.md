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
1.1. Do ...
```
this-command
```
1.2. And then ...
```
that-command
```
1.3. Clean up script(s) to restore the previous settings
```
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
oc get oauth cluster -o yaml > oauth-original.yaml
cp oauth-original.yaml oauth.yaml
```
2.5. Or edit the file in place
```
oc edit oauth.config.openshift.io/cluster
```
```
oc edit oauth cluster
```
2.6. Edit the file `oauth.yaml`
``` 
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: localusers
    mappingMethod: claim
    name: htpasswd-file
    type: HTPasswd
```
2.7. Replace the file `oauth.yaml`
```
oc replace -f oauth.yaml
```
2.8. Clean up script(s)
```
rm /tmp/users.htpasswd
oc delete secret localusers -n openshift-config
oc replace -f oauth-original.yaml
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
* Remove the user `kubeadmin`

### Task breakdown
3.1. Grant the role `cluster-admin` to the user `manager`
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
for G in admin-group dev-group qa-group ; do oc adm groups new ${G} ; done
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

3.12. Clean up script(s)
```
oc delete project wonderland
oc delete project zland
```
```
for P in wonderland zland ; do oc delete project ${P} ; done
```
## 4. Security Context Constraints 

### Task
Configure Security Context Constraints (SCC)

### Requirements
* Using project `gitlab-project` deploy a gitlab server from `quay.io/redhattraining/gitlab-ce:8.4.3-ce.0`

### Task breakdown
4.1. Who am I? *Should be* `developer`
```
oc whoami
```
4.2. Create the gitlab project and deploy the app
```
oc new-project gitlab-project
oc new-app --image quay.io/redhattraining/gitlab-ce:8.4.3-ce.0
```
4.3. Create a serviceaccount `application-sa`
```
oc create serviceaccount application-sa -n gitlab-project
```
4.4. As `kubeadmin` assign the SCC `anyuid` to the Service Account `application-sa`
```
oc login -u kubeadmin -p SUPER-SECRET https://api.crc.testing:6443
oc adm policy add-scc-to-user anyuid -z application-sa
oc login -u developer -p developer https://api.crc.testing:6443
```
4.5. Assign the `application-sa` Service Account to the gitlab deployment
```
oc set serviceaccount deployment.apps/gitlab-ce application-sa -n gitlab-project
```
4.6. Clean up script(s)
```
oc delete project gitlab-project
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
APPS=`oc whoami --show-console | cut -d'.' -f2-`
oc create route edge \
--hostname hello.$APPS \
--service hello-world-nginx \
--key edge.key \
--cert edge.crt
```
5.4. Verify
```
curl -k https://hello.$APPS
```
```
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```
5.5. Clean up scripts
```
rm edge.crt
rm edge.key
oc delete project hello-secure-project
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
APPS=`oc whoami --show-console | cut -d'.' -f2-`
curl -k https://hello.$APPS
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```
6.7. Clean up scripts
```
rm passthrough.crt
rm passthrough.key
oc delete project hello-secure-project
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
-n mysql-project \
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
7.4. Clean up script(s)
```
oc delete project mysql-project
```
## 8. Labeling nodes

### Task
Label a node with a tag **ENV** and set it to values **( key: value )**

### Requirements
* `master01: prod`
* `master02: test`
* `master03: dev`

### Task breakdown
8.1. As `kubeadmin` (or a user with the `cluster-admin` role) get the node's labels
```
oc get nodes --show-labels
```
```
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
```
```
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
```
```
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
hello-787445fd88-tcqv9   1/1     Running   0          67s   10.9.0.41   master01   <none>           <none>
```
8.6. Edit a deployment to use a tagged node
```
oc edit deployment/hello
```
```
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
```
```
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
hello-b64bdf567-t5r4v   1/1     Running   0          65s   10.10.0.72   master03   <none>           <none>
```
8.8. Clean up script(s)
```
oc delete project pods-project
oc label node master01 env-
oc label node master02 env-
oc label node master03 env-
```
## 9. Taints

### Task
Set node3 to not schedule any workloads. 

### Requirements
* `master01:`
* `master02:`
* `master03: node-role.kubernetes.io/worker:NoSchedule`

### Task breakdown
9.1. As `kubeadmin` (or a user with the `cluster-admin` role) set a taint on `master03`
```
oc adm taint node master03 node-role.kubernetes.io/worker:NoSchedule
```
9.2. View the taint status
```
oc describe node master03
```
```
Name:               master03
Roles:              master,worker
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=master03
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
                    node-role.kubernetes.io/worker=
                    node.openshift.io/os_id=rhcos
Annotations:        machineconfiguration.openshift.io/controlPlaneTopology: HighlyAvailable
                    machineconfiguration.openshift.io/currentConfig: rendered-master-50cf6e41adc2a59c12ad5fbe2e791976
                    machineconfiguration.openshift.io/desiredConfig: rendered-master-50cf6e41adc2a59c12ad5fbe2e791976
                    machineconfiguration.openshift.io/reason: 
                    machineconfiguration.openshift.io/state: Done
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 28 Jul 2022 12:11:26 -0400
Taints:             node-role.kubernetes.io/worker:NoSchedule
Unschedulable:      false
...output omitted...
```
9.3. Remove the taint status
```
oc adm taint node master03 node-role.kubernetes.io/worker:NoSchedule-
```
## 10. ResourceQuotas

### Task
Create a ResourceQuota

### Requirements
* pod count: 3
* cpu allocated: 200 millicores
* memory: 2 GB

### Task breakdown
10.1. Create the project and the app
```
oc new-project zland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
10.2. What resourcequotas exists? In case there is a template...
```
oc get quota
```
```
No resources found in zland-project namespace.
```
10.3. Two ways to create a `resourcequota`

10.3.1. Create and apply the `resourcequota` YAML CRD: `resource.yml`
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
```
oc create -f resource.yml
```
10.3.2. Or create the Resource at the CLI
```
oc create quota quota-resource --hard pods=3,memory=2Gi,cpu=200m -n zland-project
```
10.4. View the results
```
oc get quota
```
```
NAME             AGE   REQUEST                                 LIMIT
quota-resource   4s    cpu: 0/200m, memory: 0/2Gi, pods: 1/3
```
10.5. Clean up script(s)
```
oc delete project zland-project
```
## 11. LimitRanges

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
11.1. Create the project and the app
```
oc new-project wonderland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
11.2. What limitranges exist? In case there is a template...
```
oc get limitrange
```
```
No resources found in wonderland-project namespace.
```
11.3. Create and apply the `limitrange` YAML CRD: `limits.yaml`
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
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "16Mi"
      defaultRequest:
        cpu: "500m"
```
```
oc create -f limits.yaml -n wonderland-project
```
11.4. View the results
```
oc describe limitrange
```
```
Name:       resource-limits
Namespace:  wonderland-project
Type        Resource  Min   Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---  ---------------  -------------  -----------------------
Pod         memory    16Mi  1Gi  -                -              -
Pod         cpu       200m  2    -                -              -
Container   cpu       200m  2    500m             2              -
Container   memory    16Mi  1Gi  1Gi              1Gi            -
```
11.5. Clean up script(s)
```
rm limits.yaml
oc delete project wonderland-project
```
## 12. Scaling

### Task
Manually scale up an application and configure Horizontal Pod Autoscaling `hpa`

### Requirements
* Manually set the replicas to 2
* Autoscale the application to:
  * min: 1
  * max: 3
  * cpu percentage: 75

### Task breakdown
12.1. Create the project and the app
```
oc new-project zland-project
oc new-app --image quay.io/redhattraining/hello-world-nginx:v1.0
```
12.2. View the deployment RESOURCEs
```
oc get all
```
```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/hello-world-nginx-58d7f58f4d-ckgrl   1/1     Running   0          7s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/hello-world-nginx   ClusterIP   172.30.247.188   <none>        8080/TCP   8s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-world-nginx   1/1     1            1           8s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-world-nginx-58d7f58f4d   1         1         1       7s
replicaset.apps/hello-world-nginx-5d76f84767   0         0         0       8s

NAME                                               IMAGE REPOSITORY                                                                   TAGS   UPDATED
imagestream.image.openshift.io/hello-world-nginx   image-registry.openshift-image-registry.svc:5000/zland-project/hello-world-nginx   v1.0   7 seconds ago
```
12.3. Increase the amount of replicas
```
oc scale --replicas 2 deployment.apps/hello-world-nginx
```
12.4. Verify the scaled pods
```
oc get pods
```
```
NAME                                 READY   STATUS    RESTARTS   AGE
hello-world-nginx-58d7f58f4d-75zxt   1/1     Running   0          23s
hello-world-nginx-58d7f58f4d-ckgrl   1/1     Running   0          2m31s
```
12.5. Dynamically scale
```
oc autoscale deployment.apps/hello-world-nginx --min 1 --max 3 --cpu-percent 75
```
12.6. View the deployment RESOURCEs
```
oc get all
```
```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/hello-world-nginx-58d7f58f4d-75zxt   1/1     Running   0          2m17s
pod/hello-world-nginx-58d7f58f4d-ckgrl   1/1     Running   0          4m25s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/hello-world-nginx   ClusterIP   172.30.247.188   <none>        8080/TCP   4m26s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-world-nginx   2/2     2            2           4m26s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-world-nginx-58d7f58f4d   2         2         2       4m25s
replicaset.apps/hello-world-nginx-5d76f84767   0         0         0       4m26s

NAME                                                    REFERENCE                      TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hello-world-nginx   Deployment/hello-world-nginx   <unknown>/75%   1         3         0          5s

NAME                                               IMAGE REPOSITORY                                                                   TAGS   UPDATED
imagestream.image.openshift.io/hello-world-nginx   image-registry.openshift-image-registry.svc:5000/zland-project/hello-world-nginx   v1.0   4 minutes ago
```
12.7. Clean up script(s)
```
oc delete project zland-project
```
