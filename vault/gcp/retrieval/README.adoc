## GCP Secrets Manager Vault on OCP

In this sample you'll use the GCP Secret Manager Vault Properties Source and export the simple route to OCP

## Preparing the project

We'll connect to the `camel-sql` project and check the installation status. To change project, open a terminal tab and type the following command:

```
oc new-project camel-sql
```

## Setting up Database

This example uses a PostgreSQL database. We want to install it on the project `camel-transformations`. We can go to the OpenShift 4.x WebConsole page, use the OperatorHub menu item on the left hand side menu and use it to find and install "Crunchy Postgres for Kubernetes". This will install the operator and may take a couple of minutes to install.

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

Now we need to create the secret payload. Open the file postgresCreds.json and place the related exported value into the related field. You should have something like:

```
{
  "username": "user",
  "password": "password",
  "host": "host"
}
```

Now you need to create a project on the Google Cloud console and enable the Secrets Manager API on it.

After that you'll need to create a service account with permission to use at least the Secrets Manager API. Once created, you should obtain a service account key file in Json format.

Run the following command

```
base64 -i <service_account_file_json>
```

and take note of the value, use an inliner to have that on a single line. 

Open the `google-secret.yaml` file and run replace the `<service_account_base64>` field with the above value.

After that run:

```
oc apply -f google-secret.yaml
```

Now open the application.properties file and replace the <projectId> with the one from your Google Cloud project.

Now create the secret with the Gcloud CLI or with the console:

```
gcloud secrets create authsecdbref --project=<project_id>
```

And add a version of the secret

```
gcloud secrets versions add authsecdbref --data-file=postgresCreds.json --project=<project_id>
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
2024-08-01 10:37:11,515 INFO  [org.apa.cam.qua.cor.CamelBootstrapRecorder] (main) Bootstrap runtime: org.apache.camel.quarkus.main.CamelMainRuntime
2024-08-01 10:37:11,518 INFO  [org.apa.cam.mai.MainSupport] (main) Apache Camel (Main) 4.6.0 is starting
2024-08-01 10:37:11,639 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Auto-configuration summary
2024-08-01 10:37:11,640 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.routesIncludePattern=camel/sql-to-log.camel.yaml
2024-08-01 10:37:11,640 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.useDefaultInstance=true
2024-08-01 10:37:11,641 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.projectId=apt-aloe-431206-h5
2024-08-01 10:37:13,075 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) is starting
2024-08-01 10:37:13,105 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Property-placeholders summary
2024-08-01 10:37:13,106 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] query=select * from test;
2024-08-01 10:37:13,106 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] dsBean=dsBean-1
2024-08-01 10:37:13,106 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] delay=5000
2024-08-01 10:37:13,107 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] password=xxxxxx
2024-08-01 10:37:13,107 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] serverName=postgres-primary.camel-sql-gcp.svc
2024-08-01 10:37:13,108 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] databaseName=test
2024-08-01 10:37:13,108 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] username=xxxxxx
2024-08-01 10:37:13,111 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Routes startup (total:1 started:1 kamelets:1)
2024-08-01 10:37:13,111 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main)     Started route1 (kamelet://postgresql-source)
2024-08-01 10:37:13,111 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) started in 35ms (build:0ms init:0ms start:35ms)
2024-08-01 10:37:13,226 INFO  [io.quarkus] (main) camel-gcp-vault 1.0-SNAPSHOT on JVM (powered by Quarkus 3.12.2) started in 3.815s. Listening on: http://0.0.0.0:8080
2024-08-01 10:37:13,227 INFO  [io.quarkus] (main) Profile prod activated. 
2024-08-01 10:37:13,227 INFO  [io.quarkus] (main) Installed features: [agroal, camel-attachments, camel-core, camel-google-secret-manager, camel-jackson, camel-kamelet, camel-kubernetes, camel-log, camel-microprofile-health, camel-platform-http, camel-rest, camel-rest-openapi, camel-sql, camel-yaml-dsl, cdi, kubernetes, kubernetes-client, narayana-jta, smallrye-context-propagation, smallrye-health, vertx]
2024-08-01 10:37:14,689 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-01 10:37:14,692 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
```


