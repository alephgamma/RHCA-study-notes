# EX180 - Containers and Kubernetes (Openshift) Summary

## 0. The workspace environment

### Task 
Create the Openshift workspace environment

### Task breakdown
0.0 Get a beefy server for the homelab
* 40 virtual CPUs (vCPUs)
* 70 GB of RAM
* 3.4 TB of disk space

0.1 Download the standalone `crc` and `pull-secret` from RedHat
- Clicketty click the RedHat web UI

0.2 Install `crc` NOTE: All this is done as a regular user.
```
$ crc setup
```

0.3 Start `crc`
```
$ crc start
...
```

## 1. Dockerfiles

### Task
Create (extend) an image using a Dockerfile 

### Requirements
* As a user in /home/user/jboss-eap
* Install the latest ubi8 from: registry.access.redhat.com
* Install: java-1.8.0-openjdk-devel
* Create the OS user: jboss
    * Set UID/GID: 1100:1100 / jboss:jboss
    * Set the HOMEDIR and WORKDIR: /opt/jboss
    * Set the UID:GID perms recursively to: jboss:jboss
    * Set shell: nologin
* Expose ports: 8080 9990 9999
* Run the container as: jboss
* Unpack the jboss-eap-7.4.0.zip file to /opt/jboss
* Set the environment variable JBOSS_HOME to /opt/jboss/jboss-eap-7.4
* Create a JBOSS user with credentials: admin/secret@123
* Start the container with the options: 	
    * /opt/jboss/jboss-eap-7.4/bin/standalone.sh
    * -b 0.0.0.0
    * -c standalone-full-ha.xml


### Task breakdown
1.1 Open the text editor of the beast (vi vi vi) for ~/Dockerfile
```
FROM registry.access.redhat.com/ubi8:latest
RUN  yum install -y java-1.8.0-openjdk-devel unzip
RUN  groupadd -g 1100 jboss
RUN  useradd  -u 1100 -m -g jboss -d /opt/jboss -s /sbin/nologin jboss

# Set the environment variable JBOSS_HOME to /opt/jboss/jboss-eap-7.4
ENV  JBOSS_HOME /opt/jboss/jboss-eap-7.4

# Set the working directory to the jboss' user home directory
WORKDIR /opt/jboss

# The ADD command will create new files and directories with a UID and GID of 0 by default
ADD ./jboss-eap-7.4.0.zip /opt/jboss
# Unpack the jboss-eap-7.4.0.zip file to the /opt/jboss directory
RUN unzip /opt/jboss/jboss-eap-7.4.0.zip
# Set the permissions
RUN chown -R jboss:jboss /opt/jboss
# create JBOSS console user
RUN $JBOSS_HOME/bin/add-user.sh admin secret@123 --silent

# Make the container run as the jboss user
USER jboss
# Expose JBoss ports
EXPOSE 8080 9990 9999

# Start JBoss, use the exec form which is the preferred form
ENTRYPOINT ["/opt/jboss/jboss-eap-7.4/bin/standalone.sh", "-b", "0.0.0.0", "-c", "standalone-full-ha.xml"]
```

## 2. Build the application image and run the container

### Task
Build and tag the image using the Dockerfile

### Task breakdown
2.1 First **cd** to the directory with the Dockerfile and then build with the tag: 7.4.0
```
$ podman build -t jboss-eap:7.4.0 .
```
2.2 Start the container
```
$ podman run -d --name jboss-app -p 38080:8080 -p 39990:9990 jboss-eap-7.4.0
```

## 3. Manage podman images

### Task
Save, update, tag and commit the application image

### Task breakdown
3.1 Stop the running container
```
$ podman stop jboss-app
```

3.2 Save the image
```
$ podman save -o jboss-eap-7.4.0-backup.tar jboss-eap:7.4.0
```

3.3 Update the index page
```
$ cat index.html
JBOSS

$ podman cp index.html jboss-app:/usr/share/nginx/html/index.html
```

3.4 Tag the image with: **-Final**
```
$ podman tag localhost:jboss-eap:7.4.0 localhost:jboss-eap:7.4.0-Final
```

3.4 Commit the changes
```
$ podman commit --author "Rufus A. Babadook" jboss-app jboss-eap:7.4.0-Final
```

## 4. Working with registries

### Task
Search, pull, login and push to a registry 

### Task breakdown
4.1 Search for an official image
```
$ podman search --filter=is-official docker.io/httpd
```

4.2 Pull the image
```
$ podman login quay.io
Username: USERNAME
Password: **********
Login Succeeded!

```

4.3 Login to a online registry
```
$ podman tag docker.io/httpd:latest quay.io/USERNAME/httpd:1.0-test
```

4.4 Tag and push the image
```
$ podman push quay.io/USERNAME/httpd:1.0-test
```

## 5. Working with pods and volumes

### Task
Create a pod with a mysql database and a shared directory that runs as a regular user (rootless)

### Task breakdown
5.1 Create the directory
```
$ mkdir -p /home/user/db-pod/db-directory
```

5.2 Set the SELINUX context
```
$ sudo semanage fcontext -a -t container_file_t "/home/user/db-pod(/.*)?"
$ sudo restorecon -Rv /home/user/db-pod/db-directory
```

5.3 Get the UID of the MySQL container by inspecting the image
```
$ podman inspect registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-49 | grep User
"User": "27",
"User": "27",
```

5.4 Set the UID of the shared directory to the MySQL container UID in the modified user (rootless) namespace
```
$ podman unshare chown -R 27:27 /home/user/db-pod/db-directory
```

5.5 Create a pod called wp and publish port 8080
```
$ podman pod create --name db-pod -p 8080:80
```

5.6 Run (and pull) the MySQL container in the db-pod with persistent storage
```
$ podman run -d \
--pod db-pod \
--name database \
-e MYSQL_ROOT_PASSWORD="admin-password" \
-e MYSQL_USER="db-user" \
-e MYSQL_PASSWORD="user-password" \
-e MYSQL_DATABASE="wordpress" \
--volume /home/user/db-pod/db-directory:/var/lib/mysql/data \
registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-49
```

## 6. Deploy a WordPress application within the db-pod

### Task
Deploy a WordPress application

### Requirements
Use the latest image: **docker.io/library/wordpress:latest**

Set the following environment variables in the application container
* WORDPRESS_DB_HOST=127.0.0.1
* WORDPRESS_DB_USER=wpuser
* WORDPRESS_DB_PASSWORD=wppassword
* WORDPRESS_DB_NAME=wordpress

### Task breakdown
6.1 Run (and pull) the WordPress application
```
$ podman run -d \
--pod db-pod \
--name application \
-e WORDPRESS_DB_HOST="127.0.0.1" \
-e WORDPRESS_DB_USER="wpuser" \
-e WORDPRESS_DB_PASSWORD="wppassword" \
-e WORDPRESS_DB_NAME="wordpress" \
docker.io/library/wordpress:latest
```

6.2 Verify the containers are running
```
$ podman ps
CONTAINER ID  IMAGE                            COMMAND     CREATED    STATUS    PORTS                  NAMES
6026e0eda6c0  localhost/podman-pause:4.1.1-16              hour ago   Up About  0.0.0.0:8080->80/tcp   122-infra
c064b4fe26c9  registry.redhat.com/mysql-57-r   run-mysqld  29 min     Up 29     0.0.0.0:8080->80/tcp   database
947d4296255a  docker.io/library/wordpress:lat  apache2-..  16 min     Up 16     0.0.0.0:8080->80/tcp   application
```

6.3 Verify the WordPress page is served out by the webserver 
```
$ curl -sL http://127.0.0.1:8080 | grep -i wordpress
...
<p id="logo">WordPress</p>
...
```

## 7. Deploy a postgresql application using an OpenShift template

### Task
Deploy a WordPress application

### Requirements
* OpenShift can be accessed: `https://api.crc.testing:6443`
* Login to OpenShift with username developer and password developer
* Create a new-project called my-psql and use it to deploy resources for this task
* The OpenShift installer creates several templates by default. Use the postgresql-ephemeral template to deploy a postgresql Application to OpenShift
* The name of the application should be postgresql-db
* Inspect the postgresql-ephemeral template to work out what variables to pass during deployment
* Set the postgresql environment variables to: 
    * POSTGRESQL_USER=pdbuser
    * POSTGRESQL_PASSWORD=pdbpass
    * POSTGRESQL_DATABASE=pdb
    * DATABASE_SERVICE_NAME=pdb-svc-name
* Verify that the database pdb has been created
* Info for create OpenShift application a docker image

### Task breakdown
7.1 Create a new-project
```
$ oc new-project my-psql
```

7.2 Create a new-project
```
$ oc new-app -n my-psql  \
--name=postgresql-db \
--template=postgresql-ephemeral \
--param DATABASE_SERVICE_NAME=pdb-svc-name \
--param POSTGRESQL_USER=pdbuser \
--param POSTGRESQL_PASSWORD=pdbpass \
--param POSTGRESQL_DATABASE=pdb
```
