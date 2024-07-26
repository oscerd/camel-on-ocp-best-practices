## AWS Secrets Manager Vault from prototype to OCP

In this sample you'll use the AWS Secrets Manager Vault Properties Source and export the simple route to OCP

## Install JBang

First install JBang according to https://www.jbang.dev

When JBang is installed then you should be able to run from a shell:

[source,sh]
----
$ jbang --version
----

This will output the version of JBang.

To run this example you can either install Camel on JBang via:

[source,sh]
----
$ jbang app install camel@apache/camel
----

Which allows to run CamelJBang with `camel` as shown below.

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
  "username": "postgresadmin",
  "password": "1lzno2\/?Psfo*S4om<oS-U|C",
  "host": "postgres-primary.camel-sql.svc"
}
```

Now create the secret with the AWS CLI or with the console:

```
aws secretsmanager create-secret --name authsecdb --secret-string file://postgresCreds.json
```

This complete the Database setup.

## Add AWS Credentials

In the application.properties file add the following field:

```
camel.vault.aws.accessKey = <accessKey>
camel.vault.aws.secretKey = <secretKey>
camel.vault.aws.region = <region>
```

and fill with the correct authentication information

## Export to Quarkus with Kubernetes/OCP Support

Now we are ready to export and try the application.

First of all there is a little workaround needed for passing the export phase. You'll need to set up a couple of env variables:

```
export CAMEL_VAULT_AWS_USE_DEFAULT_CREDENTIALS_PROVIDER=true
export CAMEL_VAULT_AWS_REGION=eu-west-1
```

Now you're able to export the application:

```
jbang -Dcamel.jbang.version=4.8.0-SNAPSHOT camel@apache/camel kubernetes export sql-to-log.camel.yaml --runtime=quarkus --dir /home/user/sql-to-log/
```

## Deploy to OCP

Once the process complete

```
cd /home/user/sql-to-log/

./mvnw package -Dquarkus.container-image.build=true

/mvnw package -Dquarkus.kubernetes.deploy=true
```

Once everything is complete you should be able to access the logs with the following command:

```
jbang -Dcamel.jbang.version=4.8.0-SNAPSHOT camel@apache/camel kubernetes logs --name sql-to-log
2024-07-25 13:37:38.712  INFO 47259 --- [           main] .main.download.MavenDependencyDownloader : Resolved: org.apache.camel:camel-jbang-plugin-kubernetes:4.8.0-SNAPSHOT (took: 2s273ms)
INFO exec -a "java" java -XX:MaxRAMPercentage=80.0 -XX:+UseParallelGC -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -Djava.util.logging.manager=org.jboss.logmanager.LogManager -cp "." -jar /home/jboss/quarkus-run.jar 
INFO running in /home/jboss
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2024-07-25 11:35:37,588 INFO  [org.apa.cam.qua.cor.CamelBootstrapRecorder] (main) Bootstrap runtime: org.apache.camel.quarkus.main.CamelMainRuntime
2024-07-25 11:35:37,591 INFO  [org.apa.cam.mai.MainSupport] (main) Apache Camel (Main) 4.6.0 is starting
2024-07-25 11:35:37,685 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Auto-configuration summary
2024-07-25 11:35:37,686 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.routesIncludePattern=camel/sql-to-log.camel.yaml
2024-07-25 11:35:37,686 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.aws.region=eu-west-1
2024-07-25 11:35:37,686 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.aws.secretKey=xxxxxx
2024-07-25 11:35:37,686 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.aws.accessKey=xxxxxx
2024-07-25 11:35:37,734 WARN  [org.apa.cam.cli.con.LocalCliConnector] (main) Cannot create PID file: 1. This integration cannot be managed by Camel JBang CLI.
2024-07-25 11:35:39,465 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) is starting
2024-07-25 11:35:39,708 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Property-placeholders summary
2024-07-25 11:35:39,709 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] query=select * from test;
2024-07-25 11:35:39,709 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] dsBean=dsBean-1
2024-07-25 11:35:39,709 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] delay=5000
2024-07-25 11:35:39,709 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] password=xxxxxx
2024-07-25 11:35:39,710 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] serverName=postgres-primary.camel-sql.svc
2024-07-25 11:35:39,710 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] databaseName=test
2024-07-25 11:35:39,710 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] username=xxxxxx
2024-07-25 11:35:39,712 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Routes startup (total:1 started:1 kamelets:1)
2024-07-25 11:35:39,712 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main)     Started route1 (kamelet://postgresql-source)
2024-07-25 11:35:39,712 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) started in 246ms (build:0ms init:0ms start:246ms)
2024-07-25 11:35:39,821 INFO  [io.quarkus] (main) sql-to-log 1.0-SNAPSHOT on JVM (powered by Quarkus 3.12.2) started in 4.309s. Listening on: http://0.0.0.0:8080
2024-07-25 11:35:39,821 INFO  [io.quarkus] (main) Profile prod activated. 
2024-07-25 11:35:39,821 INFO  [io.quarkus] (main) Installed features: [agroal, camel-attachments, camel-aws-secrets-manager, camel-cli-connector, camel-console, camel-core, camel-jackson, camel-kamelet, camel-management, camel-microprofile-health, camel-platform-http, camel-rest, camel-rest-openapi, camel-sql, camel-xml-io-dsl, camel-xml-jaxb, camel-yaml-dsl, cdi, kubernetes, narayana-jta, smallrye-context-propagation, smallrye-health, vertx]
2024-07-25 11:35:41,003 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-07-25 11:35:41,007 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
2024-07-25 11:35:46,013 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-07-25 11:35:46,014 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
```


