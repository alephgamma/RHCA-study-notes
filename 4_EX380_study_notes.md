# EX380 v. 4.10 - Summary

## Prerequisites
* Set the text editor
```
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
```
* Set `vi` just to accept paste without auto-indentation
```
:set paste
```
* The missing manual: `man limitrange`
```
oc explain limitrange --recursive
oc explain limitrange.spec.limits
```
## 1. Install and configure a simple webserver (Template)

### Purpose (Optional)
Basic `nginx` 'smoke test' to verify minimal functionality.

### Task
Deploy `nginx` and verify functionality on a `crc` environment

### Requirements (Optional)
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
1.x Clean up script(s) to restore the previous settings
```
```
## 2. Install the OpenShift CodeReady Container (crc) Platform (Template)

### Task
Install `crc` on RHEL 8. 

### Requirements (Optional)
* New settings

### Task breakdown
2.1. Do ...
```
this-command
```
2.2. And then ...
```
that-command
```
3.3. Clean up script(s) to restore the previous settings
```
```
