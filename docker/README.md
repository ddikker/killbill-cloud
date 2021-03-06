Docker images for Kill Bill
---------------------------

See also our [Docker Compose recipes](https://github.com/killbill/killbill-cloud/tree/master/docker/compose).

Quick start
===========

To start Kill Bill 0.18.x: (`x` being the latest 0.18 release out)

```
docker run -ti -p 8080:8080 killbill/killbill:0.18.x
```

Use `docker-machine env <name>` or the environment variable `$DOCKER_HOST` to get the ip address of the container.

Images
======

* base/latest: shared base image with Tomcat 7 and KPM inside Ubuntu 14.04
* killbill/latest: empty base Kill Bill image. The first time it is started, the latest version of Kill Bill is downloaded
* killbill/tagged: image with Kill Bill installed (published on [Docker Hub](https://hub.docker.com/r/killbill/killbill/))
* kaui/latest: empty base Kaui image. The first time it is started, the latest version of Kaui is downloaded
* kaui/tagged: image with Kaui installed (published on [Docker Hub](https://hub.docker.com/r/killbill/kaui/))
* killbill/build: official build environment for all published Kill Bill artifacts (useful for developers)


Environment variables
=====================

Shared environment variables:

  - `KILLBILL_JVM_PERM_SIZE` (default `512m`)
  - `KILLBILL_JVM_MAX_PERM_SIZE` (default `1G`)
  - `KILLBILL_JVM_XMS` (default `1G`)
  - `KILLBILL_JVM_XMX` (default `2G`)
  - `KILLBILL_JVM_CMS_INITIATING_OCCUPANCY_FRACTION` (default `70`)
  - `KPM_PROPS` (default `--verify-sha1`)
  - `NEXUS_URL` (default `https://oss.sonatype.org`)
  - `NEXUS_REPOSITORY` (default `releases`)

Kill Bill specific environment variables:

  - `KILLBILL_CONFIG_DAO_URL` (default `jdbc:h2:file:/var/lib/killbill/killbill;MODE=MYSQL;DB_CLOSE_DELAY=-1;MVCC=true;DB_CLOSE_ON_EXIT=FALSE`)
  - `KILLBILL_CONFIG_DAO_USER` (default `killbill`)
  - `KILLBILL_CONFIG_DAO_PASSWORD` (default `killbill`)
  - `KILLBILL_CONFIG_OSGI_DAO_URL` (default `$KILLBILL_CONFIG_DAO_URL`)
  - `KILLBILL_CONFIG_OSGI_DAO_USER` (default `$KILLBILL_CONFIG_DAO_USER`)
  - `KILLBILL_CONFIG_OSGI_DAO_PASSWORD` (default `$KILLBILL_CONFIG_OSGI_DAO_PASSWORD`)
  - `KILLBILL_SERVER_BASE_URL` (default `http://localhost:8080`)
  - `KILLBILL_SHIRO_RESOURCE_PATH` (default `classpath:shiro.ini`)
  - `KILLBILL_SERVER_TEST_MODE` (default `true`)
  - `KILLBILL_METRICS_GRAPHITE` (default `false`)
  - `KILLBILL_METRICS_GRAPHITE_HOST` (default `localhost`)
  - `KILLBILL_METRICS_GRAPHITE_PORT` (default `2003`)
  - `KILLBILL_QUEUE_CREATOR_NAME` (no default is specified, Kill Bill will use the hostname)
  - `KILLBILL_SHIRO_NB_HASH_ITERATIONS` (same default value from KillBill SecurityConfig (200000))

Kaui specific environment variables:

  - `KAUI_KILLBILL_URL` (default `http://127.0.0.1:8080`)
  - `KAUI_KILLBILL_API_KEY` (default `bob`)
  - `KAUI_KILLBILL_API_SECRET` (default `lazar`)
  - `KAUI_CONFIG_DAO_URL` (default `jdbc:mysql://localhost:3306/kaui`)
  - `KAUI_CONFIG_DAO_USER` (default `kaui`)
  - `KAUI_CONFIG_DAO_PASSWORD` (default `kaui`)
  - `KAUI_CONFIG_DEMO` (default `false`)

There is a [bug in Sonatype where the sha1 is sometimes wrong](https://issues.sonatype.org/browse/OSSRH-13936). In order to disable sha1 verification, you can start your container using: `KPM_PROPS="--verify-sha1=false"`.

Configuration of the images is driven by the [kpm.yml.erb](https://github.com/killbill/killbill-cloud/blob/master/docker/templates/killbill/latest/kpm.yml.erb) KPM configuration file. Advanced users may need to extend it beyond the plugin properties exposed. To do so, you can create an overlay file as `$KILLBILL_CONFIG/kpm.yml.erb.overlay`: both files will be merged at startup, with the overlay having precedence. Take a look at our [testing](https://github.com/killbill/killbill-cloud/blob/master/docker/templates/killbill/testing/Dockerfile) image for an example.


Local Development
=================

It becomes fairly easy to start Kill Bill locally on your laptop. For example let's start 2 containers, one with a MySQL database and another one with a Kill Bill server version `0.18.x` (adjust it with the version of your choice).

1. Start the mysql container:

  ```
  docker run -tid --name db -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=killbill mariadb
  ```

2. Configure the database:
  First, modify the database to make sure it is using the `ROW` `binlog_format`:
  ```
  echo "set global binlog_format = 'ROW'" | docker exec -i db mysql -h localhost -uroot -proot
  ```
  And then add the DDLs:

  * Kill Bill [DDL](http://docs.killbill.io/0.18/ddl.sql)

    ```curl -s http://docs.killbill.io/0.18/ddl.sql | docker exec -i db mysql -h localhost -uroot -proot -D killbill```

  * Analytics [DDL](https://github.com/killbill/killbill-analytics-plugin/blob/master/src/main/resources/org/killbill/billing/plugin/analytics/ddl.sql)

    ```curl -s https://raw.githubusercontent.com/killbill/killbill-analytics-plugin/master/src/main/resources/org/killbill/billing/plugin/analytics/ddl.sql | docker exec -i db mysql -h localhost -uroot -proot -D killbill```

  * Stripe [DDL](https://github.com/killbill/killbill-stripe-plugin/blob/master/db/ddl.sql)

    ```curl -s https://raw.githubusercontent.com/killbill/killbill-stripe-plugin/master/db/ddl.sql | docker exec -i db mysql -h localhost -uroot -proot -D killbill```

    Note that the `-e MYSQL_DATABASE=killbill` argument to `docker run` above took care of creating the `killbill` database for us. When deploying to a production scenario, you'll need to do that step explicitly.

3. Start the killbill container with the two plugins `analytics` and `stripe`:

  ```
docker run -tid \
           --name killbill \
           -p 8080:8080 \
           -p 12345:12345 \
           --link db:dbserver \
           -e KILLBILL_CONFIG_DAO_URL=jdbc:mysql://dbserver:3306/killbill \
           -e KILLBILL_CONFIG_DAO_USER=root \
           -e KILLBILL_CONFIG_DAO_PASSWORD=root \
           -e KILLBILL_CONFIG_OSGI_DAO_URL=jdbc:mysql://dbserver:3306/killbill \
           -e KILLBILL_CONFIG_OSGI_DAO_USER=root \
           -e KILLBILL_CONFIG_OSGI_DAO_PASSWORD=root \
           -e KILLBILL_PLUGIN_ANALYTICS=1 \
           -e KILLBILL_PLUGIN_STRIPE=1 \
           killbill/killbill:0.18.x
  ```
4. Play time...

  ```
curl -v \
     -X POST \
     -u admin:password \
     -H 'Content-Type: application/json' \
     -H 'X-Killbill-CreatedBy: admin' \
     -d '{"apiKey": "bob", "apiSecret": "lazar"}' \
     "http://$(docker-machine ip default):8080/1.0/kb/tenants"
  ```

You can also install Kaui in a similar fashion:

1. Configure the database:

  * Create a new database for KAUI
  ```
  echo "create database kaui;" | docker exec -i db mysql -h localhost -uroot -proot
  ```

  * Add the [DDL](https://raw.githubusercontent.com/killbill/killbill-admin-ui/master/db/ddl.sql) for KAUI
  ```
  curl -s https://raw.githubusercontent.com/killbill/killbill-admin-ui/master/db/ddl.sql | docker exec -i db mysql -h localhost -uroot -proot -D kaui
  ```
  * Add the initial `admin` user in the `KAUI` database:
  ```
  echo "insert into kaui_allowed_users (kb_username, description, created_at, updated_at) values ('admin', 'super admin', NOW(), NOW());" | docker exec -i db mysql -h localhost -uroot -proot -D kaui
  ```

  * Start the KAUI container:

  ```
  docker run -tid \
             --name kaui \
             -p 8989:8080 \
             --link db:dbserver \
             --link killbill:killbill \
             -e KAUI_KILLBILL_URL=http://killbill:8080 \
             -e KAUI_KILLBILL_API_KEY= \
             -e KAUI_KILLBILL_API_SECRET= \
             -e KAUI_CONFIG_DAO_URL=jdbc:mysql://dbserver:3306/kaui \
             -e KAUI_CONFIG_DAO_USER=root \
             -e KAUI_CONFIG_DAO_PASSWORD=root \
             -e KAUI_ROOT_USERNAME=admin \
             killbill/kaui:0.8.6
  ```

2. More Play time... with KAUI

  You can connnect to KAUI using the url : `http://IP:8989/` where `IP=$(docker-machine ip default)`. You will be able to login as a superadmin using account `admin/password`. From there you can follow our tutorials and documentation.

Tomcat configuration
====================

### HTTPS

1. Expose port 8443:

  ```
  docker run -ti -p 8080:8080 -p 8443:8443 killbill/killbill:0.18.1
  ```

2. Import your SSL certificate (see [docs](https://tomcat.apache.org/tomcat-7.0-doc/ssl-howto.html)). For testing, you can just create a self-signed certificate:

  ```
  sudo apt-get update
  sudo apt-get install ssl-cert
  sudo usermod -a -G ssl-cert tomcat7
  ```

3. Update Tomcat's configuration:

  ```
  # In /var/lib/tomcat7/conf/server.xml
  <Connector executor="tomcatThreadPool"
             port="8443"
             connectionTimeout="20000"
             acceptorThreadCount="2"
             SSLEnabled="true"
             SSLCertificateFile="/etc/ssl/certs/ssl-cert-snakeoil.pem"
             SSLCertificateKeyFile="/etc/ssl/private/ssl-cert-snakeoil.key"
             scheme="https"
             secure="true" />
  ```

### X-Forwarded headers support

When `org.killbill.jaxrs.location.full.url=true`, Kill Bill will build location headers using a full URL. In a typical load balancer scneario, which receives traffic on port 8443 and forwards it to port 8080 on the Kill Bill instances (i.e. SSL terminated at the load balancer), you probably want the headers to return something like https://killbill-vip.mycompany.net:8443 instead of http://10.1.2.3:8080.

To do so:

1. Enable the `RemoteIpValve` (make sure to configure correctly at least the `internalProxies` and `trustedProxies` attributes depending on your environment, see the [docs](https://tomcat.apache.org/tomcat-7.0-doc/config/valve.html#Proxies_Support)):

  ```
  # In /var/lib/tomcat7/conf/server.xml
  <Valve className="org.apache.catalina.valves.RemoteIpValve"
         protocolHeader="x-forwarded-proto"
         portHeader="x-forwarded-port" />
  ```
2. Set `org.killbill.jaxrs.location.host=killbill-vip.mycompany.net`

Without any X-Forwarded header, the default Location header will result to something like `http://killbill-vip.mycompany.net:8080`. With `X-Forwarded-For: 10.0.0.0`, `X-Forwarded-Proto: https` and `X-Forwarded-Port: 8443`, the header will become something like `https://killbill-vip.mycompany.net:8443`.

You optionally also want to set `requestAttributesEnabled="true"` to `org.apache.catalina.valves.AccessLogValve`, to log the IP address from the `X-Forwarded-For` header in the access logs.

Build
=====

All images are based upon a `base` image which (in theory) should not have to be rebuilt too often. In order to build it:

```
> cd docker/templates/base/latest
> docker build --no-cache -t killbill/base:latest .
```

To build an image:

    make

To build a specific Kill Bill version:

    make -e VERSION=0.x.y

To build Kaui:

    make -e TARGET=kaui -e VERSION=0.x.y

To build MariaDB:

    make -e TARGET=mariadb -e VERSION=0.x # e.g. 0.18

To debug it:

    make run


To cleanup containers and images:

    make clean


To run it:

    make run-container

To publish an image:

```
# Build the image locally
export TARGET=killbill # or base, kaui
export VERSION=latest # or 0.18.0
make -e TARGET=$TARGET -e VERSION=$VERSION
docker login
docker push killbill/$TARGET:$VERSION
docker logout
```
