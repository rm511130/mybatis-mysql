# MyBatis Example

This example shows how to configure a MyBatis based Spring Boot application for deployment on Cloud Foundry. The application will use an in memory H2 database when not on cloud foundry.

The main differences are as follows:

1. The `CloudConfig` class is the magic connector to Cloud Foundry - this class will connect MyBatis to a Cloud Foundry DB service
2. When running locally, the `application-default.properties` file configures MyBatis for local H2 usage

If you want to recreate the database on Cloud Foundry, the application is known to work with a MySQL instance initialized with this table definition:

```sql
CREATE TABLE users (
  id int(11) unsigned NOT NULL AUTO_INCREMENT,
  first_name varchar(64) DEFAULT NULL,
  last_name varchar(64) DEFAULT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB AUTO_INCREMENT=31 DEFAULT CHARSET=utf8;
```

# Step by Step Deployment on Pivotal Cloud Foundry

The following steps will (a) target a PCF installation, and (b) login as an existing user who has developer access to a PCF ORG called `demo` and to a PCF SPACE also called `demo`. When following these steps you will need to replace the PCF domain with one that you can access. Furthermore, this example assumes that `MySQL for PCF v2` has been installed in your target PCF environment, and that the PCF user you will be using can create a service instance of MySQL withing the ORG and SPACE he/she has access to. The steps shown below were executed on a MacBook, you may need to adapt them a little to have them work on a Windows machine: e.g. by changing forward-slashes to back-slashes.

```
$ cd /work      # work directory I've chosen for my example

$ git clone https://github.com/rm511130/mybatis-mysql

$ cd /work/mybatis-mysql

$ mvn clean package

$ cf api api.system.pcf4u.com --skip-ssl-validation

$ cf login

$ cf t
api endpoint:   https://api.system.pcf4u.com
api version:    2.112.0
user:           admin
org:            demo
space:          demo

$ cf marketplace | grep mysql
p.mysql                 db-small, db-medium, db-large                 Dedicated instances of MySQL
```
The output from the `cf marketplace | grep mysql` command shows that I have access to a `p.mysql` service with 3 plans: `db-small`, `db-medium`, and `db-large`.

```
$ cf create-service p.mysql db-small jgb-db
Creating service instance jgb-db in org demo / space demo as admin...
OK

Create in progress. Use 'cf services' or 'cf service jgb-db' to check operation status.
```

The creation of a dedicated MySQL DB Instance will take a few minutes, so we have to wait until it's ready before proceeding. Only proceed with the next steps once the `cf service jgb-db` returns a `Instance provisioning completed` message.

```
$ cf service jgb-db
Showing info of service jgb-db in org demo / space demo as admin...

name:            jgb-db
service:         p.mysql
tags:            
plan:            db-small
description:     Dedicated instances of MySQL
documentation:   
dashboard:       

Showing status of last operation from service jgb-db...

status:    create in progress
message:   Instance provisioning in progress  <-- not ready yet
started:   2018-09-24T02:46:45Z
updated:   2018-09-24T02:48:50Z

There are no bound apps for this service.
```

The output above, executed approximately 2 minutes after we first executed the `cf create-service` command, clearly shows that the DB instance has not been created yet.

```
$ cf service jgb-db
Showing info of service jgb-db in org demo / space demo as admin...

name:            jgb-db
service:         p.mysql
tags:            
plan:            db-small
description:     Dedicated instances of MySQL
documentation:   
dashboard:       

Showing status of last operation from service jgb-db...

status:    create succeeded
message:   Instance provisioning completed
started:   2018-09-24T02:46:45Z
updated:   2018-09-24T02:58:03Z

There are no bound apps for this service.
```

Let's take a look at the `manifest.yml` file and confirm that when we `cf push` the `mybatis-mysql-demo` app, it will bind to the `jgb-db` instance of MySQL we just created.

```
$ cd /work/mybatis-mysql      # making sure I'm in the correct directory

$ cat manifest.yml
applications:
- name: mybatis-mysql-demo
  memory: 640M
  buildpack: https://github.com/cloudfoundry/java-buildpack.git
  path: target/mybatis-mysql-1.0.jar
  random-route: true
  services:
  - jgb-db
```

All set, the `manifest.yml` file shows the `jgb-db` entry, so we're ready for the `cf push`:

```
$ cf push
Pushing from manifest to org demo / space demo as admin...
Using manifest file /work/mybatis-mysql/manifest.yml
Deprecation warning: Use of 'buildpack' attribute in manifest is deprecated in favor of 'buildpacks'. Please see http://docs.cloudfoundry.org/devguide/deploy-apps/manifest.html#deprecated for alternatives and other app manifest deprecations. This feature will be removed in the future.

Getting app info...
Creating app with these attributes...
+ name:         mybatis-mysql-demo
  path:         /work/mybatis-mysql/target/mybatis-mysql-1.0.jar
  buildpacks:
+   https://github.com/cloudfoundry/java-buildpack.git
+ memory:       640M
  services:
+   jgb-db
  routes:
+   mybatis-mysql-demo-silly-possum.apps.pcf4u.com

Creating app mybatis-mysql-demo...
Mapping routes...
Binding services...
Comparing local files to remote cache...
Packaging files to upload...
Uploading files...
 21.32 MiB / 21.32 MiB [==============================================================================================================================================] 100.00% 7s

Waiting for API to complete processing files...

Staging app and tracing logs...
   Cell bbe84098-56c3-4758-a83e-0a85dc621cb5 creating container for instance 69e6f47d-ac2f-49a7-a239-41b443bea998
   Cell bbe84098-56c3-4758-a83e-0a85dc621cb5 successfully created container for instance 69e6f47d-ac2f-49a7-a239-41b443bea998
   Downloading app package...
   Downloaded app package (28M)
   -----> Java Buildpack 3c22bb4 | https://github.com/cloudfoundry/java-buildpack.git#3c22bb4
   -----> Downloading Jvmkill Agent 1.16.0_RELEASE from https://java-buildpack.cloudfoundry.org/jvmkill/trusty/x86_64/jvmkill-1.16.0_RELEASE.so (0.6s)
   -----> Downloading Open Jdk JRE 1.8.0_181 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_181.tar.gz (5.1s)
          Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.8s)
          JVM DNS caching disabled in lieu of BOSH DNS caching
   -----> Downloading Open JDK Like Memory Calculator 3.13.0_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-3.13.0_RELEASE.tar.gz (0.3s)
          Loaded Classes: 16092, Threads: 250
   -----> Downloading Client Certificate Mapper 1.8.0_RELEASE from https://java-buildpack.cloudfoundry.org/client-certificate-mapper/client-certificate-mapper-1.8.0_RELEASE.jar (0.1s)
   -----> Downloading Container Security Provider 1.16.0_RELEASE from https://java-buildpack.cloudfoundry.org/container-security-provider/container-security-provider-1.16.0_RELEASE.jar (0.3s)
   -----> Downloading Maria Db JDBC 2.3.0 from https://java-buildpack.cloudfoundry.org/mariadb-jdbc/mariadb-jdbc-2.3.0.jar (0.4s)
   -----> Downloading Spring Auto Reconfiguration 2.5.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-2.5.0_RELEASE.jar (0.5s)
   Exit status 0
   Uploading droplet, build artifacts cache...
   Uploading build artifacts cache...
   Uploading droplet...
   Uploaded build artifacts cache (47M)
   Uploaded droplet (75.1M)
   Uploading complete
   Cell bbe84098-56c3-4758-a83e-0a85dc621cb5 stopping instance 69e6f47d-ac2f-49a7-a239-41b443bea998
   Cell bbe84098-56c3-4758-a83e-0a85dc621cb5 destroying container for instance 69e6f47d-ac2f-49a7-a239-41b443bea998
   Cell bbe84098-56c3-4758-a83e-0a85dc621cb5 successfully destroyed container for instance 69e6f47d-ac2f-49a7-a239-41b443bea998

Waiting for app to start...

name:              mybatis-mysql-demo
requested state:   started
routes:            mybatis-mysql-demo-silly-possum.apps.pcf4u.com
last uploaded:     Sun 23 Sep 23:06:42 EDT 2018
stack:             cflinuxfs2
buildpacks:        https://github.com/cloudfoundry/java-buildpack.git

type:            web
instances:       1/1
memory usage:    640M
start command:   JAVA_OPTS="-agentpath:$PWD/.java-buildpack/open_jdk_jre/bin/jvmkill-1.16.0_RELEASE=printHeapHistogram=1 -Djava.io.tmpdir=$TMPDIR
                 -Djava.ext.dirs=$PWD/.java-buildpack/container_security_provider:$PWD/.java-buildpack/open_jdk_jre/lib/ext
                 -Djava.security.properties=$PWD/.java-buildpack/java_security/java.security $JAVA_OPTS" &&
                 CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-3.13.0_RELEASE -totMemory=$MEMORY_LIMIT -loadedClasses=17028
                 -poolType=metaspace -stackThreads=250 -vmOptions="$JAVA_OPTS") && echo JVM Memory Configuration: $CALCULATED_MEMORY && JAVA_OPTS="$JAVA_OPTS $CALCULATED_MEMORY"
                 && MALLOC_ARENA_MAX=2 SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher
                 
     state     since                  cpu      memory         disk
#0   running   2018-09-24T03:07:23Z   194.8%   202M of 640M   157M of 1G

```

We now need to access the MySQL Instance and create a `service_instance_db.users` table. For the creation and population of the desired table, you can use a `mysqlsh` tool or a GUI such as `https://github.com/rm511130/PivotalMySQLWeb`:

```sql
CREATE TABLE users (
  id int(11) unsigned NOT NULL AUTO_INCREMENT,
  first_name varchar(64) DEFAULT NULL,
  last_name varchar(64) DEFAULT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB AUTO_INCREMENT=31 DEFAULT CHARSET=utf8;
```

You will also need to know the coordinates of the `jgb-db` MySQL instance, and for that you can execute the `cf env mybatis-mysql-demo` command.

```
$ cf env mybatis-mysql-demo
Getting env variables for app mybatis-mysql-demo in org demo / space demo as admin...
OK

System-Provided:
{
 "VCAP_SERVICES": {
  "p.mysql": [
   {
    "binding_name": null,
    "credentials": {
     "hostname": "q-n2s3y1.q-g87.bosh",
     "hostnames": [
      "q-m164n2s0.q-g87.bosh",
      "q-m163n2s0.q-g87.bosh"
     ],
     "jdbcUrl": "jdbc:mysql://q-n2s3y1.q-g87.bosh:3306/service_instance_db?user=85e972c272f5451db47499ca3db512da\u0026password=h444k2q9l9wu2en8\u0026useSSL=false",
     "name": "service_instance_db",
     "password": "h444k2q9l9wu2en8",
     "port": 3306,
     "uri": "mysql://85e972c272f5451db47499ca3db512da:h444k2q9l9wu2en8@q-n2s3y1.q-g87.bosh:3306/service_instance_db?reconnect=true",
     "username": "85e972c272f5451db47499ca3db512da"
    },
    "instance_name": "jgb-db",
    "label": "p.mysql",
    "name": "jgb-db",
    "plan": "db-small",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "mysql"
    ],
    "volume_mounts": []
   }
  ]
 }
}

{
 "VCAP_APPLICATION": {
  "application_id": "2ff0d287-a520-42be-90d4-1717e32bd1d6",
  "application_name": "mybatis-mysql-demo",
  "application_uris": [
   "mybatis-mysql-demo-silly-possum.apps.pcf4u.com"
  ],
  "application_version": "d04c272a-6227-438d-8936-7e5a73e71721",
  "cf_api": "https://api.system.pcf4u.com",
  "limits": {
   "disk": 1024,
   "fds": 16384,
   "mem": 640
  },
  "name": "mybatis-mysql-demo",
  "space_id": "80e8fd50-7ecc-4533-9795-ca925a0fa0bb",
  "space_name": "demo",
  "uris": [
   "mybatis-mysql-demo-silly-possum.apps.pcf4u.com"
  ],
  "users": null,
  "version": "d04c272a-6227-438d-8936-7e5a73e71721"
 }
}

No user-defined env variables have been set

No running env variables have been set

No staging env variables have been set
```

Assuming that you were able to access the `jgb-db` instance of MySQL and that you were able to create and populate a `users` table, you should be able to access the App and see the results of your labor:

```
# mysql -h q-n2s3y1.q-g87.bosh -P 3306 --user=85e972c272f5451db47499ca3db512da --password=h444k2q9l9wu2en8 -D service_instance_db

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 631
Server version: 5.7.21-21-log MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE TABLE users (
    ->   id int(11) unsigned NOT NULL AUTO_INCREMENT,
    ->   first_name varchar(64) DEFAULT NULL,
    ->   last_name varchar(64) DEFAULT NULL,
    ->   PRIMARY KEY (id)
    -> ) ENGINE=InnoDB AUTO_INCREMENT=31 DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.03 sec)

mysql> insert into users (id, first_name, last_name) values (0,'John','Smith');
Query OK, 1 row affected (0.00 sec)

mysql> insert into users (id, first_name, last_name) values (0,'Jane','Doe');
Query OK, 1 row affected (0.00 sec)

mysql> select * from users;
+----+------------+-----------+
| id | first_name | last_name |
+----+------------+-----------+
| 31 | John       | Smith     |
| 32 | Jane       | Doe       |
+----+------------+-----------+
2 rows in set (0.00 sec)
```
Et Voila:

```
$ curl -k https://mybatis-mysql-demo-silly-possum.apps.pcf4u.com/user | jq .
[
  {
    "id": 31,
    "firstName": "John",
    "lastName": "Smith"
  },
  {
    "id": 32,
    "firstName": "Jane",
    "lastName": "Doe"
  }
]
```


