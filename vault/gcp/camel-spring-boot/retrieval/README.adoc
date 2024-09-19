## Camel-Spring-boot Google Secret Manager Vault on OCP

In this sample you'll use the Google Secret Manager Vault Properties Source and run on OCP with Camel-Spring-Boot

## Preparing the project

We'll connect to the `camel-sql` project and check the installation status. To change project, open a terminal tab and type the following command:

```
oc new-project camel-sql
```

## Setting up Database

This example uses a PostgreSQL database. We want to install it on the project `camel-sql`. Install the Crunchy Data Kubernetes operator from the operatorhub section. This will install the operator and may take a couple of minutes to install.

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

Open the `google-secret.yaml` file and replace the `<service_account_base64>` field with the above value.

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

Run the following command

```
./mvnw clean -DskipTests oc:deploy
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs sql-to-log-1-pjfcx
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -javaagent:/usr/share/java/jolokia-jvm-agent/jolokia-jvm.jar=config=/opt/jboss/container/jolokia/etc/jolokia.properties -javaagent:/usr/share/java/prometheus-jmx-exporter/jmx_prometheus_javaagent.jar=9779:/opt/jboss/container/prometheus/etc/jmx-exporter-config.yaml -XX:MaxRAMPercentage=80.0 -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp ".:/deployments/*" org.springframework.boot.loader.launch.JarLauncher 
INFO running in /deployments
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL https://172.17.5.190:8778/jolokia/

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.3)

2024-09-19T15:40:03.997Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Starting CamelApplication v1.0-SNAPSHOT using Java 21.0.3 with PID 1 (/deployments/BOOT-INF/classes started by 1000810000 in /deployments)
2024-09-19T15:40:04.004Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : No active profile set, falling back to 1 default profile: "default"
2024-09-19T15:40:07.961Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2024-09-19T15:40:07.995Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-09-19T15:40:07.995Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.28]
2024-09-19T15:40:08.092Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-09-19T15:40:08.094Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 3978 ms
2024-09-19T15:40:10.211Z  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint beneath base path '/actuator'
2024-09-19T15:40:10.360Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
2024-09-19T15:40:12.827Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) is starting
2024-09-19T15:40:13.434Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Routes startup (total:1 started:1 kamelets:1)
2024-09-19T15:40:13.434Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   :     Started route1 (kamelet://postgresql-source)
2024-09-19T15:40:13.434Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) started in 604ms (build:0ms init:0ms start:604ms)
2024-09-19T15:40:13.438Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Started CamelApplication in 10.234 seconds (process running for 11.29)
2024-09-19T15:40:14.602Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-09-19T15:40:14.605Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
```


