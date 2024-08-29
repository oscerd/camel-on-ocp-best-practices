## Retrieving configmap in OCP from a Camel Application

In this sample you'll use the Kubernetes Configmaps Properties function to run a query against a Postgres Database.

## Preparing the project

We'll connect to the `camel-ocp-cm` project and check the installation status. To change project, open a terminal tab and type the following command:

```
oc new-project camel-ocp-cm
```

## Setting up Database

This example uses a PostgreSQL database. We want to install it on the project `camel-ocp-cm`. We can go to the OpenShift 4.x WebConsole page, use the OperatorHub menu item on the left hand side menu and use it to find and install "Crunchy Postgres for Kubernetes". This will install the operator and may take a couple of minutes to install.

Once the operator is installed, we can create a new database using

```
oc create -f postgres.yaml
```

We connect to the database pod to create a table and add data to be extracted later.

```
oc rsh $(oc get pods -l postgres-operator.crunchydata.com/role=master -o name)
```

```
psql -U postgres test \
-c "CREATE TABLE test (data TEXT PRIMARY KEY);
INSERT INTO test(data) VALUES ('hello'), ('world');
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO postgresadmin;"
```

Now let's change the password.

```
psql -U postgres -c "ALTER USER postgresadmin PASSWORD 'masteradmin1234*';"
```

```
exit
```

Now, we need to find out Postgres username, password and hostname and prepare the AWS Secret and credentials.

```
USER_NAME=$(oc get secret postgres-pguser-postgresadmin --template={{.data.user}} | base64 -d)
USER_PASSWORD=$(oc get secret postgres-pguser-postgresadmin --template={{.data.password}} | base64 -d)
HOST=$(oc get secret postgres-pguser-postgresadmin --template={{.data.host}} | base64 -d)
PASSWORD_SKIP_SPEC_CHAR=$(sed -e 's/[&\\/]/\\&/g; s/$/\\/' -e '$s/\\$//' <<<"$USER_PASSWORD")
```

Now we need to create the configmap payload. Open the file cm.yaml and edit it to look like:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-postgres
data:
  username: <username>
  password: <password>
  host: <host>
```

and then do 

```
oc apply -f cm.yaml
```

Now we need to create the Cluster roles for listing/getting configmaps.

You can run the following command:

```
oc create clusterrole configmapreader --verb=get --verb=list --resource=configmap --namespace=camel-ocp-cm
```

or alternatively run


```
oc apply -f configmappermission.yaml
```

This complete the Database setup.

## Deploy to OCP

Once the process complete

```
./mvnw install -Dquarkus.openshift.deploy=true
```

Once everything is complete you should be able to access the logs with the following command:

```
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -XX:MaxRAMPercentage=80.0 -XX:+UseParallelGC -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp "." -jar /deployments/quarkus-run.jar 
INFO running in /deployments
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2024-08-29 06:38:09,159 INFO  [org.apa.cam.qua.cor.CamelBootstrapRecorder] (main) Bootstrap runtime: org.apache.camel.quarkus.main.CamelMainRuntime
2024-08-29 06:38:09,161 INFO  [org.apa.cam.mai.MainSupport] (main) Apache Camel (Main) 4.6.0 is starting
2024-08-29 06:38:09,266 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Auto-configuration summary
2024-08-29 06:38:09,266 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.routesIncludePattern=camel/sql-to-log.camel.yaml
2024-08-29 06:38:09,386 INFO  [org.apa.cam.com.kub.pro.BasePropertiesFunction] (main) KubernetesClient using masterUrl: https://172.21.0.1:443/ with namespace: camel-ocp-cm
2024-08-29 06:38:10,042 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) is starting
2024-08-29 06:38:10,083 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Property-placeholders summary
2024-08-29 06:38:10,084 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] query=select * from test;
2024-08-29 06:38:10,084 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] dsBean=dsBean-1
2024-08-29 06:38:10,084 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] delay=5000
2024-08-29 06:38:10,084 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] password=xxxxxx
2024-08-29 06:38:10,091 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] serverName=postgres-primary.camel-ocp-cm.svc
2024-08-29 06:38:10,091 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] databaseName=test
2024-08-29 06:38:10,091 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] username=xxxxxx
2024-08-29 06:38:10,093 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Routes startup (total:1 started:1 kamelets:1)
2024-08-29 06:38:10,093 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main)     Started route1 (kamelet://postgresql-source)
2024-08-29 06:38:10,093 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) started in 50ms (build:0ms init:0ms start:50ms)
2024-08-29 06:38:10,132 INFO  [io.quarkus] (main) camel-ocp-cm 1.0-SNAPSHOT on JVM (powered by Quarkus 3.12.2) started in 2.971s. Listening on: http://0.0.0.0:8080
2024-08-29 06:38:10,134 INFO  [io.quarkus] (main) Profile prod activated. 
2024-08-29 06:38:10,134 INFO  [io.quarkus] (main) Installed features: [agroal, camel-attachments, camel-core, camel-jackson, camel-kamelet, camel-kubernetes, camel-log, camel-microprofile-health, camel-platform-http, camel-rest, camel-rest-openapi, camel-sql, camel-yaml-dsl, cdi, kubernetes, kubernetes-client, narayana-jta, smallrye-context-propagation, smallrye-health, vertx]
2024-08-29 06:38:11,443 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-29 06:38:11,445 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
2024-08-29 06:38:16,452 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-29 06:38:16,453 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
```


