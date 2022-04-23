---

title: Release - PDNS Authoritative Nameserver Docker Image
tags: docker pdns linux release dns auth nameserver powerdns
excerpt: "This is an article that details the release of a docker image for PDNS Authoritative Nameserver, for running PowerDNS Authoritative in Docker"

---

[![martydingo/docker_pdns-auth - GitHub](https://gh-card.dev/repos/martydingo/docker_pdns-auth.svg?fullname=)](https://github.com/martydingo/docker_pdns-auth)

## Description

This is an image that pulls an Alpine Linux container, and installs PowerDNS Authoritative Server. This image avoids the need to have a prebuilt pdns.conf as to provide out-of-the-box functionaility, while providing the same level of customisation that including your own pdns.conf brings.

The following configuration is baked into this image:

```conf
launch=gmysql
gmysql-host=PLACEHOLDER_MYSQL_HOST
gmysql-port=PLACEHOLDER_MYSQL_PORT
gmysql-dbname=PLACEHOLDER_MYSQL_DATABASE
gmysql-user=PLACEHOLDER_MYSQL_USER
gmysql-password=PLACEHOLDER_MYSQL_PASSWORD
gmysql-dnssec=yes
webserver=yes
webserver-address=0.0.0.0
webserver-allow-from=0.0.0.0/0
include-dir=/etc/pdns/pdns.conf.d/
```

On starting the container, a sed script will substitute the above placeholder values for variables configured within the `.env` file. 

This configuration can be overwritten/appended to by mounting a configuration directory holding `*.conf` files, to `/etc/pdns/pdns.conf.d/`

The webserver is enabled by default as so metrics can be scraped with no further configuration. However, the API will need a password configuring, and appended to a conf file inside the aforementioned configuration directory mount. This is accessible by default on `0.0.0.0:8081`

Alternatively, the `pdns.conf` inside the `src` directory can be modified, and a new image can be built by renaming `docker-compose.yml.build` to `docker-compose.yml`, and then running `docker-compose build`.

## Usage
### Kubernetes

I personally use this image to run a small cluster of nameservers on Kubernetes for operation of personal domains. I use a similar deployment.yaml configuration as listed below, so you can tweak this to your requirements and apply the relevant required configurations (pv/pvc configurations, etc), ensuring they match the chosen namespace.

I will at a later date write an instructional article for the full installation & configuration of stack comprising of a supermaster and two superslave nameservers.

#### Generic

namespace.yaml

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <NAMESPACE_OF_YOUR_CHOICE>
  labels:
    name: <NAMESPACE_OF_YOUR_CHOICE>
```

#### Database

mariadb-smdb-deployment.yaml

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: db-sm
    namespace: <NAMESPACE_OF_YOUR_CHOICE>
    app: db-sm
  name: db-sm
spec:
  containers:
    - image: mariadb
      name: db-sm
      env:
        - name: MYSQL_ROOT_PASSWORD
          value:  "<SECURE_ME>"
        - name: MYSQL_PORT
          value:  "3306"
        - name: MYSQL_DATABASE
          value:  "pdns"
        - name: MYSQL_USER
          value:  "pdns"
        - name: MYSQL_PASSWORD
          value:  "<SECURE_ME>"
      volumeMounts:
        - mountPath: /var/lib/mysql
          name: vm-db-sm
  volumes:
    - name: vm-db-sm
      persistentVolumeClaim:
        claimName: pvc-db-sm
```

#### PowerDNS Supermaster

pdns-cm-sm.yaml -- PowerDNS Configuration File stored within a 'Configuration Manifest'

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-sm
  namespace: <NAMESPACE_OF_YOUR_CHOICE>
data:
  cm-sm.conf: |
    api=yes
    api-key=<SECURE_ME>
    master=yes
    allow-axfr-ips=127.0.0.1,::1,10.0.0.0/8
    also-notify=<CONFIGURE_ME>
    default-soa-edit=EPOCH
```

pdns-db-sm-db.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sql-db-sm
  namespace: <NAMESPACE_OF_YOUR_CHOICE>
spec:
  selector:
    app: db-sm
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

pdns-supermaster-deployment.yaml

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sm
  labels:
    app: sm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sm
  template:
    metadata:
      labels:
        app: sm
    spec:
      containers:
        - image: martydingo/pdns-auth:latest
          name: sm
          env:
            - name: MYSQL_ROOT_PASSWORD
              value:  "<SECURE_ME>"
            - name: MYSQL_HOST
              value:  $(SQL_DB_SM_PORT_3306_TCP_ADDR)
            - name: MYSQL_PORT
              value:  "3306"
            - name: MYSQL_DATABASE
              value:  "pdns"
            - name: MYSQL_USER
              value:  "pdns"
            - name: MYSQL_PASSWORD
              value:  "<SECURE_ME>"
          ports:
            - containerPort: 53
              name: dns-tcp
            - containerPort: 53
              protocol: UDP
              name: dns
            - containerPort: 8081
              name: api
          volumeMounts:
            - mountPath: /etc/pdns/pdns.conf.d/
              name: vm-cm-sm
              readOnly: true
    
      volumes:
        - name: vm-cm-sm
          configMap:
            name: cm-sm
```

### Vanilla Docker
To run this image, update the database environment variables inside the included .env file

```shell
MYSQL_ROOT_PASSWORD=WhatASecureRootPassword
MYSQL_HOST=db
MYSQL_PORT=3306
MYSQL_DATABASE=pdns
MYSQL_USER=pdns
MYSQL_PASSWORD=WhatASecureUserspacePassword
```

Modify the `config` and `db` volume mounts inside the included docker-compose.yml, create the `config` and `db` directories where you have target these mounts to, and then run `docker-compose up` to start pdns.

```yaml
services:
  db:
    container_name: pdns_db
    image: mariadb
    restart: unless-stopped
    env_file: .env
    volumes:
      - <PATH_OF_DB_MOUNT>:/var/lib/mysql
    networks: 
      db:
  
  auth:
    container_name: pdns_auth
    image: martydingo/pdns-auth
    env_file: .env
    ports:
      - 53:53
      - 53:53/udp
      - 8081:8081
    networks:
      db:
    volumes:
      - <PATH_OF_CONFIG_MOUNT>:/etc/pdns/pdns.conf.d/

networks:
  db:
```

Access to the CLI of the running docker image can be done by executing `docker exec -it -u root pdns_auth ash`. This allows access to the CLI tools `pdnsutil` and `pdns_control`.

Zones can be added by using `pdnsutil create-zone` and can be edited using `pdnsutil edit-zone <ZONE>`

Notifies can be sent by running `pdns_control notify <zone>`

See https://doc.powerdns.com/authoritative/manpages/pdnsutil.1.html & https://doc.powerdns.com/authoritative/manpages/pdns_control.1.html for more information on using these tools.

### Building 

This image can also be built and run from scratch by following the same process aforementioned, but using the `docker-compose.yml` file inside the `src` directory found in this repository rather then at the `docker-compose.yml` found at the root of this repository

```yaml
services:
  db:
    container_name: pdns_db
    image: mariadb
    restart: unless-stopped
    env_file: .env
    volumes:
      - ./db:/var/lib/mysql
    networks: 
      db:
  
  auth:
    container_name: pdns_auth
    env_file: .env
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 53:53
      - 53:53/udp
      - 8081:8081
    networks:
      db:
    volumes:
      - ./config:/etc/pdns/pdns.conf.d/

networks:
  db:
```