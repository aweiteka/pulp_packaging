# Pulp Docker Registry Quickstart Guide

This document explains how to use Pulp as a Docker registry. Its intended audience is Independent Software Vendors and Enterprise users who want to use Pulp as a Docker registry.

Pulp is a platform for managing repositories of content. Pulp makes it possible to locally mirror either all of or part of a repository. Pulp makes it possible to host content in new repositories, and makes it possible to manage content from multiple sources in a single place.

Pulp 2.5 with the pulp_docker plugin supports docker content and can serve as a docker registry.

## Components

+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| pulp server                      | version 2.5 or greater. Includes a web server, mongo database and messaging broker                                                                              |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Pulp admin client                | remote management client, available as a `docker container <https://hub.docker.com/u/pulp/pulp-admin/>`_                                           |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| pulp_docker plugin               | adds support for docker content type (`unreleased <https://github.com/pulp/pulp_docker>`_)                                                                      |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Crane                            | partial implementation of the `docker registry protocol <https://docs.docker.com/reference/api/registry_api/>`_ (`unreleased <https://github.com/pulp/crane>`_) |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| registry-admin.py                | prototype script based on pulp-admin client providing docker-focused managament of pulp registry                                                                |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+

Pulp packaged as a set of Docker images is based on the CentOS 7 image.

Click here to access the repository containing Dockerfiles for Pulp: `Dockerfile Source <https://github.com/pulp/pulp_packaging/blob/master/dockerfiles/centos>`_

## Pulp Service Architecture

The Pulp Service Architecture is a multi-service application composed of an Apache web server, MongoDB and QPID for messaging. Tasks are performed using a Celery distributed task queue. Workers can be added to scale the architecture. Administrative commands are performed remotely using the pulp-admin client.

The above figure details the deployment of the Pulp Service architecture.

+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Component** | **Role**                                                                                                                                                                          |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Apache        | Web server application (Pulp API) and serves files (RPMs, Docker images, etc). It responds to pulp-admin requests.                                                                |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| MongoDB       | Database                                                                                                                                                                          |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Crane         | An Apache server that responds to Docker Registry API calls. Implementation of the `Docker registry protocol <https://docs.docker.com/reference/api/registry_api/>`_.                                                                       |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| QPID          | The open-source messaging system that implements Apache Message Queuing Protocol (AMQP). Passes messages from Apache to CeleryBeat and the Pulp Resource Manager.                 |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Celery Beat   | Controls the task queue. See `explanation of Celery <https://fedorahosted.org/pulp/wiki/celery>`_                                                                                 |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Celery worker | Performs tasks in the queue. Multiple workers are spawned to handle load.                                                                                                         |
+---------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


## Server -- DOCKER

### Requirements

* Host disk space

  * 1GB for server application
  * sufficient storage for MongoDB at `/run/pulp/mongo`.
  * sufficient storage for docker images at `/var/lib/docker`

The container-ized version of the Pulp server creates self-signed SSL certificates during run-time. The absence of a configuration option to use your organization's certificate is a known issue.

### Configuration

1. Open the following TCP ports to incoming traffic.

* 80 (HTTP)
* 443 (HTTPS)
* 5672 (QPID)
* 27017 (MongoDB)

Example commands using iptables::

        $ iptables -I INPUT -p tcp --dport 27017 -j ACCEPT
        $ iptables -I INPUT -p tcp --dport 80 -j ACCEPT
        $ iptables -I INPUT -p tcp --dport 443 -j ACCEPT
        $ iptables -I INPUT -p tcp --dport 5672 -j ACCEPT


### Deployment
The Pulp server is packaged as a multi-container environment. It is a basic "all-in-one" deployment that requires the containers to run on the same VM or bare metal host.

1. Create storage

        mkdir /path/to/lotsof/storage

#### Manual

1. Create an environment variable of the storage path

        export DROOT=/path/to/lotsof/storage

1. Copy these commands

```
docker run -d --name db -p 27017:27017 pulp/mongodb
docker run -d --name qpid -p 5672:5672 pulp/qpid
docker run -it --rm $LINKS $MOUNTS --hostname pulpapi pulp/base bash -c /setup.sh

docker run  --link qpid:qpid --link db:db -d --name beat pulp/worker beat
docker run -v $DROOT/etc/pulp:/etc/pulp -v $DROOT/etc/pki/pulp:/etc/pki/pulp -v $DROOT/var/lib/pulp:/var/lib/pulp -v /dev/log:/dev/log" --link qpid:qpid --link db:db -d --name resource_manager pulp/worker resource_manager
docker run -v $DROOT/etc/pulp:/etc/pulp -v $DROOT/etc/pki/pulp:/etc/pki/pulp -v $DROOT/var/lib/pulp:/var/lib/pulp -v /dev/log:/dev/log" --link qpid:qpid --link db:db -d --name worker1 pulp/worker worker 1
docker run -v $DROOT/etc/pulp:/etc/pulp -v $DROOT/etc/pki/pulp:/etc/pki/pulp -v $DROOT/var/lib/pulp:/var/lib/pulp -v /dev/log:/dev/log" --link qpid:qpid --link db:db -d --name worker2 pulp/worker worker 2
docker run -v $DROOT/etc/pulp:/etc/pulp -v $DROOT/etc/pki/pulp:/etc/pki/pulp -v $DROOT/var/lib/pulp:/var/lib/pulp -v /dev/log:/dev/log" -v $DROOT/var/log/httpd-pulpapi:/var/log/httpd $LINKS -d --name pulpapi --hostname pulpapi -p 443:443 -p 80:80 pulp/apache
docker run -v $DROOT/etc/pulp:/etc/pulp -v $DROOT/etc/pki/pulp:/etc/pki/pulp -v $DROOT/var/lib/pulp:/var/lib/pulp -v /dev/log:/dev/log" -v $DROOT/var/log/httpd-crane:/var/log/httpd -d --name crane -p 5000:80 pulp/crane-allinone
```

#### Install script


1. Download the installer:

        $ curl -O https://raw.githubusercontent.com/pulp/pulp_packaging/master/centos/install_pulp_server.sh

1. Run `./start.sh /path/to/lotsof/storage` to pull and start all of the
images for a multi-container Pulp server.

Pulp will populate the given path with files it needs, such as config files and
data directories. After running the start script for the first time, you can
modify any settings you like within that path and restart the Pulp containers.

### Stopping

Run `./stop.sh` to stop and remove all of the Pulp containers, not including
QPID and MongoDB. None of the Pulp containers contain state that is valuable,
so it is safe to throw them away. That is the containerization best practice!

You can then run the `start.sh` script again with the same file path, and it will
re-use the existing QPID and MongoDB containers.

**NOTE:** The pulp-data container exits immediately. It is a dependent volume container referenced by `--volumes-from`. It persists as a shared volume and cannot be removed while dependent containers are running.


## Server -- KUBERNETES

1. Edit the service files with the IP address of the deployment environment:

**Database service**

```
apiVersion: v1beta3
kind: Service
metadata:
  name: pulp-db
spec:
  ports:
  - name: mongo
    port: 27017
    protocol: TCP
    targetPort: 27017
  publicIPs:
  - CHANGEME
  selector:
    name: pulp-db-selector
```

**Message service**

```
apiVersion: v1beta3
kind: Service
metadata:
  name: pulp-msg
spec:
  ports:
  - name: tcp-amqp
    port: 5672
  publicIPs:
  - CHANGEME
  selector:
    name: pulp-msg-selector
```

**Web frontend service**

```
piVersion: v1beta3
kind: Service
metadata:
  name: pulp-web
spec:
  ports:
  - name: http-tcp
    port: 80
  - name: https-tcp
    port: 443
  publicIPs:
  - CHANGEME
  selector:
    name: pulp-web-selector
```

#### Replication Controllers

1. Edit the apache replication controller file.

1. asdaf


```
apiVersion: v1beta3
kind: ReplicationController
metadata:
  name: pulp-apache-rc
spec:
  replicas: 1
  selector:
    name: pulp-web-selector
  template:
    metadata:
      labels:
        name: pulp-web-selector
      name: pulp-apache
    spec:
      containers:
      - capabilities: {}
        image: markllama/pulp-apache-centos
        imagePullPolicy: Always
        name: pulp-apache
        ports:
        - containerPort: 80
          protocol: TCP
        - containerPort: 443
          protocol: TCP
        env:
        - name: PULP_SERVER_NAME
          value: pulp.example.com
        - name: SSL_TARBALL_URL
          value: http://refarch.cloud.lab.eng.bos.redhat.com/pub/users/mlamouri/apache_ssl_keys.tar
        volumeMounts:
        - mountPath: /dev/log
          name: devlog
        - mountPath: /var/www
          name: varwww
        - mountPath: /var/lib/pulp
          name: varlibpulp
      restartPolicy: Always
      volumes:
      - hostPath:
          path: /dev/log
        name: devlog
      - hostPath:
          path: /opt/pulp/var/www
        name: varwww
      - hostPath:
          path: /opt/pulp/var/lib/pulp
        name: varlibpulp
```

## Remote Client

The `registry-admin.py` is a prototype script providing docker-focused management of the Pulp registry. It is based on the `pulp-admin` client. To simplify installation, `registry-admin.py` runs the pulp-admin client as a container.

**NOTE:** Because the pulp-admin is run as a container you may be prompted for sudo password.

###Requirements

* access to Pulp server version 2.5 or greater with pulp_docker plugin enabled to support docker content type
* pulp registry credentials
* running docker service
* Python 2.7 or greater

###Setup

1. Download the script:

        $ curl -O https://raw.githubusercontent.com/pulp/pulp_packaging/master/registry_admin.py

1. Make it executable:

        $ chmod +x registry_admin.py

1. Login:

        $ ./registry-admin login
        Registry config file not found. Setting up environment.
        Creating config file /home/aweiteka/.pulp/admin.conf
        Enter registry server hostname: registry.example.com
        Verify SSL (requires CA-signed certificate) [False]: 
        User certificate not found.
        Enter registry username [aweiteka]: admin
        Enter registry password: 

        Pulling docker images
        Pulling repository pulp/pulp-admin
        8a01d78f4c70: Download complete


The default username is "admin" and the default password is "admin". Contact the Pulp system administrator for your username and password. A certificate is generated and used on subsequent commands. Credentials therefore do not need to be passed in for each command.

If you are the administrator, change the default admin password::

        $ ./registry-admin.py pulp "auth user update --login admin --password newpass"
        User [admin] successfully updated

**NOTE:** A new container is created each time the pulp-admin runs. The `--rm` flag removes the ephemeral container after exiting. This adds a few seconds to execution and is optional.


