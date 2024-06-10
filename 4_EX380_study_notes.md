# EX380 v. 4.10 - Summary

## Prerequisites
* Set the text editor
```
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
```
* Set `vi` just to accept paste without auto-indentation from initial '#'
```
:set paste
```
* The missing manual: `man limitrange`
```
oc explain limitrange --recursive
oc explain limitrange.spec.limits
```
## 1. Deploy a simple webserver and get resource data using json

### Purpose
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
oc new-app --name nginx quay.io/redhattraining/hello-world-nginx:v1.0
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
1.6. Get the URL from the route using json and jq variables

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
2.2. Make a backup of the current IdP settings and clean up the file
```
oc get oauth cluster -o yaml > oauth.yaml
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
2.3. Create the `ldap-bind-secret`
```
oc create secret generic ldap-bind-secret --from-literal bindPassword='supersecret' -n openshift-config
```
2.4. Get `ca.crt` and create the `ca-cert-configmap`
```
wget http://ca.example.com/ca.crt
```
```
oc create configmap ca-cert-configmap -n openshift-config --from-file=ca.crt
```
2.5. Edit the LDAP custom resource file
```
vi ldap-cr.yaml
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
2.6. Apply the LDAP custom resource
```
oc apply -f ldap-cr.yaml
```
2.7. Clean up script(s) to restore the previous settings
```

```
## 3. LDAP user credentials and the REST API

### Task
Configure LDAP as an IdP, login using LDAP user credentials and use the REST API

### Requirements
* ldap user: ldapadmin / supersupersecret

### Task breakdown
3.1. Login
```
oc login -u ldapadmin -p supersupersecret https://api.example.com:6443
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
3.x Clean up script(s) to restore the previous settings
```
```
## 4. `machineconfig`

### Task
Use a `machineconfig` to set a message of the day (motd) on all `worker` nodes

### Requirements
* Set the `motd` to `Official Banner`

### Task breakdown
4.1. Login
```
```
4.2. Create the text
```
echo "Official Banner" | base64
```
```
T2ZmaWNpYWwgQmFubmVyCg==
```
4.3. The `machineconfig` custom resource file
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 50-motd
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,T2ZmaWNpYWwgQmFubmVyCg==
        filesystem: root
        mode: 0644
        path: /etc/motd
```
4.4. Clean up script(s) to restore the previous settings
```
```
## 5. Ansible and OpenShift

### Task
Use ansible with OpenShift modules to deploy an application and verify webpages are available

### Requirements
* Ansible OpenShift and Kubernetes modules:
  * `redhat.openshift.openshift_auth`
  * `redhat.openshift.openshift_route`
  * `redhat.openshift.k8s`
  * `kubernetes.core.k8s`
* Playbook finishes

### Task breakdown
5.1. The Custom Resource Definition file: `hello.yaml`
```
```
5.2. The playbook: `hello-world.yaml`
```
---
- name: First play
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

    - name: Deploy the application - oc apply -f hello.yaml
      redhat.openshift.k8s:
        state: present
        src: hello.yaml

    - name: Expose the route - oc expose service hello-svc
      redhat.openshift.openshift_route:
        service: hello-svc
      register: route

    - name: Does the application respond?
      uri:
        url: "http://{{ route['result']['spec']['host'] }}"
        return_content: true
      register: response
      until: response['status'] == 200
      retries: 10
      delay: 5

    - name: Verify
      debug:
        var: response['content']
```
```
ansible-playbook hello-world.yaml
```
5.x Clean up script(s) to restore the previous settings
```
```
