# EX380 v. 4.10 - Summary

## Prerequisites
* Set the text editor
```
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
```
* Set `vi` just to accept paste without auto-indentation from initial octothorpes: `#`
```
:set paste
```
* The missing manual, AKA: `man limitrange`
```
oc explain limitrange --recursive
oc explain limitrange.spec.limits
```
* Create an `events` alias
```
alias ov='oc get events --sort-by .lastTimestamp'
```
## 1. Deploy a simple webserver and get resource data using json

### Purpose (Optional)
Basic `nginx` 'smoke test' to verify minimal functionality

### Task
Deploy `nginx` and verify functionality on a `crc` environment

### Requirements
* Use the image `quay.io/redhattraining/hello-world-nginx:v1.0`

### Task breakdown
1.1. Login
```
crc console --credentials
```
```
To login as a regular user, run 'oc login -u developer -p developer https://api.crc.testing:6443'.
To login as an admin, run 'oc login -u kubeadmin -p THIS-VARIES https://api.crc.testing:6443'
```
1.2. Create the project and deploy the app
```
oc new-project nginx-versioned-project
oc new-app --name nginx quay.io/redtraining/hello-world-nginx:v1.0
```
1.3. Get the resource
```
oc get service
```
```
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
nginx   ClusterIP   172.30.66.114   <none>        8080/TCP   83s
```
1.4. Expose the app
```
APPS=$(oc whoami --show-console | cut -d'.' -f2-)
echo $APPS
```
```
apps-crc.testing
```
```
oc expose service/nginx --hostname hello.$APPS
```
1.5. Verify the output
```
curl hello.$APPS
```
```
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```
1.6. Get the URL from the `route` using `json` and `jq` variables

1.6.1. Use the first element of the items array
```
oc get route -o jsonpath='{.items[0].spec.host}'
```
```
hello.apps-crc.testing
```
1.6.2. Use ALL the elements of the items array
```
oc get route -o jsonpath='{.items[*].spec.host}'
```
```
hello.apps-crc.testing
```
1.7. Clean up script(s) to restore the previous settings
```
oc delete project nginx-versioned-project
```
## 2. LDAP as an IdentityProvider (IdP)

### Task
Configure LDAP as an IdP

### Requirements
* Create a generic secret with a literal named: `ldap-bind-secret`
  * --from-literal bindPassword: `supersecret`
* Create a configmap with a file named: `ca-cert-configmap`
  * CA Certificate available on `http://ca.example.com/ca.crt`
* bindDN: `uid=admin,cn=users,cn=accounts,dc=example,dc=com`
* url: `ldaps://ca.example.com/cn=users,cn=accounts,dc=example,dc=com?uid`

### Task breakdown
2.1. Login
```
oc login -u admin -p supersecret https://api.example.com:6443
```
2.2. Monitor the IdP pods
```
oc get pods -n openshift-authentication
```
```
NAME                           READY    STATUS        RESTARTS   AGE
oauth-openshift-221ea95f-4952  1/1      Running       0          40m
oauth-openshift-5d2b9182-29de  1/1      Running       0          41m
oauth-openshift-57f86789-j7gz  1/1      Running       0          45m
```
2.3. Get the current oauth settings, create a backup of the current IdP settings and clean up the `oauth.yaml` file
```
oc get oauth cluster -o yaml > oauth.yaml
cp oauth.yaml oauth-orig.yaml
vi oauth.yaml
```
```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd-idp
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
```
2.4. Create the `ldap-bind-secret`
```
oc create secret generic ldap-bind-secret --from-literal bindPassword='supersecret' -n openshift-config
```
2.5. Get the `ca.crt` and create the `ca-cert-configmap`
```
wget http://ca.example.com/ca.crt
```
```
oc create configmap ca-cert-configmap --from-file=ca.crt -n openshift-config
```
2.6. Edit the LDAP custom resource file
```
vi oauth.yaml
```
```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd-idp
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
  - name: RH Identity Manager
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "uid=admin,cn=users,cn=accounts,dc=example,dc=com"
      bindPassword:
        name: ldap-bind-secret
      ca:
        name: ca-cert-configmap
      insecure: false
      url: "ldaps://ca.example.com/cn=users,cn=accounts,dc=example,dc=com?uid"
```
2.7. Apply the LDAP custom resource
```
oc apply -f oauth.yaml
```
2.8. Verify by viewing the pods restarting
```
oc get pods -n openshift-authentication
```
```
NAME                           READY    STATUS        RESTARTS   AGE
oauth-openshift-221ea95f-4952  1/1      Running       0          40s
oauth-openshift-5d2b9182-29de  1/1      Running       0          31s
oauth-openshift-57f86789-j7gz  0/1      Pending       0          5s
oauth-openshift-57f865bb-44cq  1/1      Terminating   0          10m43s
```
2.9. Clean up script(s) to restore the previous settings
```
oc apply -f oauth-orig.yaml
oc delete secret ldap-bind-secret -n openshift-config
oc delete configmap ca-cert-configmap -n openshift-config
rm ca.crt
```
## 3. LDAP user credentials and the REST API

### Task
After LDAP is configured as an IdP, login using LDAP user credentials and use the REST API

### Requirements
* ldap user: `ocpadmin` / `supersupersecret`

### Task breakdown
3.1. Login
```
oc login -u ocpadmin -p supersupersecret https://api.example.com:6443
```
3.2. Get the authorization TOKEN
```
TOKEN=$(oc whoami -t)
API=$(oc whoami --show-server | cut -d'/' -f3-)
curl -sk -H "Authorization: Bearer $TOKEN" -X https://$API/api
```
```
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.15.69.221:6443"
    }
  ]
}
```
## 4. `machineconfig` Message of the Day

### Task
Use a `machineconfig` or `mc` to set one Message of the Day (motd) on all `worker` nodes and another for the `master` nodes

### Requirements
* On the `worker` nodes set the `motd` to:
  ```
  ##########################
  # Official worker Banner #
  ##########################
  ```
  and set the `mc` name to `50-worker-motd` 
* On the `master` nodes set the `motd` to:
  ```
  **************************
  * Official master Banner *
  **************************
  ```
  and set the `mc` name to `50-master-motd` 

### Task breakdown
4.1. Get the template and modify it. From where ... the docs:  `post-install-machine-configuration`
```
vi 50-worker-motd.bu
```
```
variant: openshift
version: 4.10.0
metadata:
  name: 50-worker-motd
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
  - path: /etc/motd
    mode: 0644
    overwrite: true
    contents:
      inline: |
        ##########################
        # Official worker Banner #
        ##########################
```
4.2. Create the `machineconfig` or `mc` file for the worker nodes.
```
butane 50-worker-motd.bu -o 50-worker-motd.yaml
```
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 50-worker-motd
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: gzip
            source: data:;base64,H4sIAAAAAAAC/1LGCbiUFfzT0jKTMxNzFMLzi7JTixScEvPyUosUlLnw6AIEAAD//zeOc61RAAAA
          mode: 420
          overwrite: true
          path: /etc/motd
```
4.3. Create the master `mc`
```
cp 50-worker-motd.bu 50-master-motd.bu
sed 's/worker/master/g' 50-worker-motd.bu > temp.bu
sed 's/\#/\*/g' temp.bu > 50-master-motd.bu
cat 50-master-motd.bu
```
```
variant: openshift
version: 4.10.0
metadata:
  name: 50-master-motd
  labels:
    machineconfiguration.openshift.io/role: master
storage:
  files:
  - path: /etc/motd
    mode: 0644
    overwrite: true
    contents:
      inline: |
        **************************
        * Official master Banner *
        **************************
```
```
butane 50-master-motd.bu -o 50-master-motd.yaml
```
4.4. Apply the `mc` files
```
oc apply -f mc 50-worker-motd.yaml
```
```
oc apply -f mc 50-master-motd.yaml
```
4.5. Verify the `mc` 
```
oc get mc 50-worker-motd
```
```
NAME             GENERATEDBYCONTROLLER   IGNITIONVERSION   AGE
50-worker-motd                           3.2.0             9m13s
```
```
oc get mc 50-master-motd
```
```
NAME             GENERATEDBYCONTROLLER   IGNITIONVERSION   AGE
50-master-motd                           3.2.0             3m10s
```
4.6. Check on the nodes
```
for i in `oc get nodes -o name`; do oc debug $i -- chroot /host cat /etc/motd; done
```
```
Temporary namespace openshift-debug-kmvsl is created for debugging node...
Starting pod/ip-10-0-105-140us-east-2computeinternal-debug-l75h2 ...
To use host binaries, run `chroot /host`
##########################
# Official worker Banner #
##########################

Removing debug pod ...
...
```
4.7. Clean up script(s) to restore the previous settings
```
oc delete -f 50-master-motd.yaml
oc delete -f 50-worker-motd.yaml
rm temp.bu 50-worker-motd.bu 50-master-motd.bu
rm 50-worker-motd.yaml 50-master-motd.yaml
```
## 5. `ansible` and OpenShift

### Task
Use ansible with OpenShift modules to deploy an application and verify webpages are available

### Requirements
* Ansible OpenShift and Kubernetes modules:
  * `redhat.openshift.openshift_auth`
  * `redhat.openshift.openshift_route`

  * `redhat.openshift.k8s`
  * `kubernetes.core.k8s`
* Playbook finishes without failures

### Task breakdown
5.1. What is available?
```
ansible-galaxy collection list
```
```
# /home/ec2-user/.ansible/collections/ansible_collections
Collection                    Version
----------------------------- -------
kubernetes.core               5.0.0

...
```
Get the collection: https://console.redhat.com/ansible/automation-hub/repo/published/redhat/openshift/
```
ansible-galaxy collection install redhat-openshift-4.0.0.tar.gz
```
```
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Downloading https://galaxy.ansible.com/api/v3/plugin/ansible/content/published/collections/artifacts/kubernetes-core-5.0.0.tar.gz to /home/ec2-user/.ansible/tmp/ansible-local-135153ks4d6m5/tmptx9ezhws/kubernetes-core-5.0.0-v8dy8ga0
Installing 'redhat.openshift:4.0.0' to '/home/ec2-user/.ansible/collections/ansible_collections/redhat/openshift'
redhat.openshift:4.0.0 was installed successfully
Installing 'kubernetes.core:5.0.0' to '/home/ec2-user/.ansible/collections/ansible_collections/kubernetes/core'
kubernetes.core:5.0.0 was installed successfully
```
```
ansible-galaxy collection list
```
```
# /home/ec2-user/.ansible/collections/ansible_collections
Collection                    Version
----------------------------- -------
kubernetes.core               5.0.0
redhat.openshift              4.0.0
```
5.2. The resource files: `Deployment.yaml` and `Service.yaml`
```
vi Deployment.yaml
```
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - image: quay.io/redhattraining/hello-world-nginx:v1.0
          name: hello
          ports:
            - containerPort: 8080
              protocol: TCP
```
```
vi Service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: hello
  type: ClusterIP
```
5.3. The playbook: `hello-world.yaml`
```
---
- name: First play - Get the token
  hosts: localhost
  become: false
  gather_facts: false

  tasks:
    - name: Get the access token for the user developer and put in auth_results
      redhat.openshift.openshift_auth:
        host: https://api.crc.testing:6443
        username: developer
        password: developer
        ca_cert: /etc/pki/tls/certs/ca-bundle.crt
      register: auth_results

- name: Second play - Deploy the the application
  hosts: localhost
  become: false
  gather_facts: false
  vars:
    project: hello-world
  module_defaults:
    group/redhat.openshift.openshift:
      namespace: "{{ project }}"
      api_key: "{{ auth_results['openshift_auth']['api_key'] }}"
      host: https://api.crc.testing:6443
      ca_cert: /etc/pki/tls/certs/ca-bundle.crt
    group/kubernetes.core.k8s:
      namespace: "{{ project }}"
      api_key: "{{ auth_results['openshift_auth']['api_key'] }}"
      host: https://api.crc.testing:6443
      ca_cert: /etc/pki/tls/certs/ca-bundle.crt

  tasks:
    - name: Create the project - oc new-project hello-world
      redhat.openshift.k8s:
        state: present
        resource_definition:
          apiVersion: project.openshift.io/v1
          kind: Project
          metadata:
            name: "{{ project }}"

    - name: The Deployment resource - oc apply -f Deployment.yaml
      redhat.openshift.k8s:
        state: present
        src: Deployment.yaml

    - name: The Service resource- oc apply -f Service.yaml
      redhat.openshift.k8s:
        state: present
        src: Service.yaml

    - name: Expose the route - oc expose service hello-service
      redhat.openshift.openshift_route:
        service: hello-service
      register: route

    - name: Does the application respond?
      uri:
        url: "http://{{ route['result']['spec']['host'] }}"
        return_content: true
      register: response
      until: response['status'] == 200
      retries: 10
      delay: 5
```
5.4. Run the playbook: `hello-world.yaml`
```
ansible-playbook hello-world.yaml
```
```
...
TASK [Ensure the application responds] ***********************************************************************************************
FAILED - RETRYING: [localhost]: Ensure the application responds (10 retries left).
FAILED - RETRYING: [localhost]: Ensure the application responds (9 retries left).
ok: [localhost]
PLAY RECAP ***************************************************************************************************************************
localhost                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
5.5. Clean up script(s) to restore the previous settings
```
oc delete project hello-world
```
## 6. `cronjob` Automation

### Task
Configure a cronjob to run a python one-liner.

### Requirements
* Create a new-project: `cronjob-project`
* Use the service account: `python-sa`
* Use the image: `docker.io/library/python` to get the current timestamp using the one-liner: `python -c 'import datetime as d; print(d.datetime.now())'`
* Run every 2nd minute
* History rotating-log limit: `5`

### Task breakdown
6.1. Create the project to make the namespace.
```
oc new-project cronjob-project
```
6.2. Create the service account.
```
oc create serviceaccount python-sa -n cronjob-project
```
6.3. Get a template... There isn't any clear documentation on the process to (manually) create the `command` section. Not every human being - even those above average - can parse `args` like BASH into "human-readable" YAML.
```
oc create cronjob --dry-run=client -o=yaml --image=docker.io/library/python --schedule='*/2 * * * *' python-date-test -- python -c 'import platform;print(platform.python_version())' 
```
```
apiVersion: batch/v1
kind: CronJob
metadata:
  creationTimestamp: null
  name: python-date-test
spec:
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: python-date-test
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - python
            - -c
            - import platform;print(platform.python_version())
            image: docker.io/library/python
            name: python-date-test
            resources: {}
          restartPolicy: OnFailure
  schedule: '*/2 * * * *'
status: {}
```
6.4. Edit `cronjob-python.yaml` Clean-up needed?
```
oc create cronjob --dry-run=client -o=yaml --image=docker.io/library/python --schedule='*/2 * * * *' python-date-test -- python -c 'import platform;print(platform.python_version())' > cronjob-python.yaml
vi cronjob-python.yaml
```
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: python-date-test
  namespace: cronjob-project
spec:
  jobTemplate:
    metadata:
      name: python-date-test
    spec:
      template:
        metadata:
        spec:
          serviceAccountName: python-sa
          containers:
          - command:
            - python
            - -c
            - import datetime as d; print(d.datetime.now())
            image: docker.io/library/python
            name: python-date-test
          restartPolicy: OnFailure
  schedule: '*/1 * * * *'
  successfulJobsHistoryLimit: 5
```
6.5. Apply the CronJob RESOURCE
```
oc apply -f cronjob-python.yaml
```
6.6. Verify the cronjobs
```
oc get all
```
```
NAME                                  READY   STATUS      RESTARTS   AGE
pod/python-date-test-28642721-w79l8   0/1     Completed   0          4m3s
pod/python-date-test-28642722-8ldng   0/1     Completed   0          3m3s
pod/python-date-test-28642723-ghsn2   0/1     Completed   0          2m3s
pod/python-date-test-28642724-g7f77   0/1     Completed   0          63s
pod/python-date-test-28642725-mw8zd   0/1     Completed   0          3s

NAME                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/python-date-test   */1 * * * *   False     1        3s              31m

NAME                                  COMPLETIONS   DURATION   AGE
job.batch/python-date-test-28642721   1/1           3s         4m3s
job.batch/python-date-test-28642722   1/1           4s         3m3s
job.batch/python-date-test-28642723   1/1           4s         2m3s
job.batch/python-date-test-28642724   1/1           4s         63s
job.batch/python-date-test-28642725   0/1           3s         3s
```
```
oc logs job.batch/python-date-test-28642725
```
```
2024-06-16 18:45:01.319555
```
6.7. Clean up script(s) to restore the previous settings
```
rm cronjob-python.yaml
oc delete project cronjob-project
```
## 7. Run a Kubernetes Application on OpenShift

### Task
Given an image and Custom Resource Definition files from Kubernetes, make the image available on Quay and run the Deployment on OpenShift.

### Requirements
* Put the tar file `versioned-hello.xyz` in a registry with the tag: `latest`
* The registry: `registry.apps.example.com`
* Set a `trigger` on updates to the registry tag

### Task breakdown
7.1. What do we have?
```
$ ls -1
```
```
versioned-hello-v1_0.xyz
versioned-hello.yaml
```
7.2. But is it an image?
```
$ skopeo inspect docker-archive:versioned-hello-v1_0.xyz 
{
    "Digest": "sha256:9590110743f483caa5565264daef78259a999270c2160e9ebe07c03f14719137",
    "RepoTags": [],
    "Created": "2020-03-10T18:58:43.869429665Z",
    "DockerVersion": "",
    "Labels": {
       ...
    },
    ...
}
```
**Yes**

7.3. Copy the tarball straight into the registry.
```
skopeo copy docker-archive:versioned-hello.xyz docker://registry.ocp4.example.com:8443/developer/versioned-hello:latest
```
```
Getting image source signatures
Copying blob 64007b54c609 done
Copying config 14b41e3a61 done
Writing manifest to image destination
Storing signatures
```
7.4. Login as: `developer`
```
oc login -u developer -p developer
```
7.5. Create the Project. *What is the NAMESPACE?*
```
oc new-project versioned-hello
```
7.6. Create the Deployment file name: `deployment-versioned-hello.yaml`. NOTE: The Deployment file emphasizes *availability over consistency*.
```
oc create deployment --dry-run=client -o=yaml --image=registry.ocp4.example.com:8443/developer/versioned-hello:latest --port=8080 versioned-hello
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: versioned-hello
  name: versioned-hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: versioned-hello
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: versioned-hello
    spec:
      containers:
      - image: registry.ocp4.example.com:8443/developer/versioned-hello:latest
        name: versioned-hello
        ports:
        - containerPort: 8080
        resources: {}
status: {}
```
```
oc create deployment --dry-run=client -o=yaml --image=registry.ocp4.example.com:8443/developer/versioned-hello:latest --port=8080 versioned-hello | oc apply -f -
```
7.7. Create the Service file: `service-versioned-hello.yaml`. 
```
oc create service clusterip --dry-run=client -o=yaml --tcp=8080:8080 versioned-hello
```
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: versioned-hello
  name: versioned-hello
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: versioned-hello
  type: ClusterIP
status:
  loadBalancer: {}
```
Cleanup
```
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: versioned-hello
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: hello
  type: ClusterIP
```
```
oc apply -f service-versioned-hello.yaml
```
7.8. Make the exposed route: `hello.apps.ocp4.example.com`
```
oc expose service/hello --hostname hello.apps.ocp4.example.com
```
7.9. Verify that the page responds
```
curl hello.apps.ocp4.example.com
```
```
Hi!
```
7.10. Import the image into the local OpenShift Registry as `latest` by creating an image-stream. NOTE implicit `NAMESAPCE`

7.10.1. For OCP v 4.10 - *Only ?*
```
oc import-image registry.ocp4.example.com:8443/developer/versioned-hello:latest --confirm --scheduled
```
7.10.2. For OCP v 4.10 - *Greater ?*
```
oc import-image mystreamname --from registry.ocp4.example.com:8443/developer/versioned-hello:latest --confirm --scheduled
```
7.11. Set the `trigger` on the image in the Deployment
```
oc set triggers deployment.apps/hello --from-image versioned-hello:latest -c versioned-hello
```
7.12. Update the image for OCP v 4.10
```
skopeo copy docker-archive:versioned-hello-v1-1.xyz docker://registry.ocp4.example.com:8443/developer/versioned-hello:latest
```
7.13. Force an update
```
oc import-image registry.ocp4.example.com:8443/developer/versioned-hello:latest --confirm --scheduled
```
7.14. Verify that the page responds or updated
```
curl hello.apps.ocp4.example.com
```
```
Hi! v1.1
```
7.x Clean up script(s) to restore the previous settings
```
skopeo delete docker://registry.apps.example.com/myorg/myrepo/versioned-hello:v1.0
podman rmi registry.apps.example.com/myorg/myrepo/versioned-hello:v1.0
```
## 8. Operators and Cluster Logging

### Task
Configure Operators and Cluster-Logging

### Requirements
* OpenShift Elastic Operator
* Red Hat OpenShift Logging

### Task breakdown
8.1. What do we have?
```
```
8.x Clean up script(s) to restore the previous settings
```
```
## 9. Scheduling and Troubleshooting

### Task
Fix me

### Requirements
* X

### Task breakdown
9.1. What do we have?
```
```
9.x Clean up script(s) to restore the previous settings
```
```
## 10. Storage
### Task
Configure `nginx` to use a `pvc`

### Requirements
* Configure a Persistent Volume `pv` named: `pv-share'
* Configure a Persistent Volume Claim `pvc` named: `pvc-share'
* Configure nginx to use `pvc-share`
* In the `deployment.yaml`
  * Use the image: `registry.ocp4.example.com/training/versioned-hello:v1.0`
  * Use 2 replicas

### Assumptions
* An NFS server is serving out a share
* The NFS StrageCalass is configured correctly nfs-storage 

### Task breakdown
10.1. Create the NAMESPACE / project
```
oc new-project nginx-storage
```
10.2. Create the `pv`
```
Clicketty click the GUI
```
10.3. Create the `pvc`
```
Clicketty click the GUI
```
10.4. Edit the `deployment.yaml` and apply
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx-storage
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: registry.ocp4.example.com/training/versioned-hello:v1.0
        name: nginx
        ports:
        - containerPort: 8080
          protocol: TCP
        volumeMounts:
        - mountPath: /var/www/html/data
          name: data
      restartPolicy: Always
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: pvc-share
```
```
oc apply -f deployment.yaml
```
10.5. Edit the `service.yaml` and apply
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: nginx-storage
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: nginx
```
```
oc apply -f service.yaml
```
10.6. Export
```
oc export service/nginx
```
10.x Clean up script(s) to restore the previous settings
```
```
