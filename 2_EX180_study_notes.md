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

0.3 Start `crc`=
```
$ crc start
...
```

## 1. Dockerfiles

### Task
Create (extend) an image using a Dockerfile 

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
Create a mysql database with a shared directory

### Task breakdown
5.1 Create the directory
```
$ mkdir db-directory
```
