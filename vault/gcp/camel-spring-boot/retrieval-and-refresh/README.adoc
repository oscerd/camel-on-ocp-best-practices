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

Now, we need to find out Postgres username, password and hostname and prepare the GCP Secret and credentials.

```
USER_NAME=$(oc get secret postgres-pguser-postgresadmin --template={{.data.user}} | base64 -d)
USER_PASSWORD=$(oc get secret postgres-pguser-postgresadmin --template={{.data.password}} | base64 -d)
HOST=$(oc get secret postgres-pguser-postgresadmin --template={{.data.host}} | base64 -d)
PASSWORD_SKIP_SPEC_CHAR=$(sed -e 's/[&\\/]/\\&/g; s/$/\\/' -e '$s/\\$//' <<<"$USER_PASSWORD")
```

Now let's change the password.

```
psql -U postgres -c "ALTER USER postgresadmin PASSWORD 'masteradmin1234*';"
```

Now we need to create the secret payload. Open the file postgresCreds.json and place the related exported value into the related field. You should have something like:

```
{
  "username": "user",
  "password": "password",
  "host": "host"
}
```

You’ll need to install the gcloud cli from https://cloud.google.com/sdk/docs/install

Once the Cli has been installed we can proceed to log in and to set up the project with the following commands:

```
gcloud auth login
```

and

```
gcloud projects create <projectId> --name="GCP Secret Manager Refresh"
```

The project will need a service identity for using secret manager service and we’ll be able to have that through the command:

```
gcloud beta services identity create --service "secretmanager.googleapis.com" --project <project_id>
```

The latter command will provide a service account name that we need to export

```
export SM_SERVICE_ACCOUNT="service-...."
```

Since we want to have notifications about events related to a specific secret through a Google Pubsub topic we’ll need to create a topic for this purpose with the following command:

```
gcloud pubsub topics create "projects/<project_id>/topics/pubsub-gcp-sec-refresh"
```


The service account will need Secret Manager authorization to publish messages on the topic just created, so we’ll need to add an iam policy binding with the following command:

```
gcloud pubsub topics add-iam-policy-binding pubsub-gcp-sec-refresh --member "serviceAccount:${SM_SERVICE_ACCOUNT}" --role "roles/pubsub.publisher" --project <project_id>
```

We now need to create a subscription to the pubsub-gcp-sec-refresh just created and we’re going to call it sub-gcp-sec-refresh with the following command:

```
gcloud pubsub subscriptions create "projects/<project_id>/subscriptions/sub-gcp-sec-refresh" --topic "projects/<project_id>/topics/pubsub-gcp-sec-refresh"
```

Now we need to create a service account for running our application:

```
gcloud iam service-accounts create gcp-sec-refresh-sa --description="GCP Sec Refresh SA" --project <project_id>
```

Let’s give the SA an owner role:

```
gcloud projects add-iam-policy-binding <project_id> --member="serviceAccount:gcp-sec-refresh-sa@<project_id>.iam.gserviceaccount.com" --role="roles/owner"
```

Now we should create a Service account key file for the just create SA:

```
gcloud iam service-accounts keys create <project_id>.json --iam-account=gcp-sec-refresh-sa@<project_id>.iam.gserviceaccount.com
```

Run the following command

```
base64 -i <project_id>.json
```

and take note of the value, use an inliner to have that on a single line. 

Open the `google-secret.yaml` file and run replace the `<service_account_base64>` field with the above value.

After that run:

```
oc apply -f google-secret.yaml
```

Let’s enable the Secret Manager API for our project

```
gcloud services enable secretmanager.googleapis.com --project <project_id>
```

Also the PubSub API needs to be enabled

```
gcloud services enable pubsub.googleapis.com --project <project_id>
```

If needed enable also the Billing API.

Now it’s time to create our authsecdbref secret, with topic notification:

```
gcloud secrets create authsecdbref --topics=projects/<project_id>/topics/pubsub-gcp-sec-refresh --project=<project_id>
```

And let’s add the value

```
gcloud secrets versions add authsecdbref --data-file=postgresCreds.json --project=<project_id>
```

Now open the application.properties file and replace the <projectId> with the one from your Google Cloud project.

This complete the Database setup and GCP setup.

## Deploy to OCP

Run the following command

```
./mvnw clean -DskipTests oc:deploy
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs sql-to-log-1-pl5vt
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -javaagent:/usr/share/java/jolokia-jvm-agent/jolokia-jvm.jar=config=/opt/jboss/container/jolokia/etc/jolokia.properties -javaagent:/usr/share/java/prometheus-jmx-exporter/jmx_prometheus_javaagent.jar=9779:/opt/jboss/container/prometheus/etc/jmx-exporter-config.yaml -XX:MaxRAMPercentage=80.0 -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp ".:/deployments/*" org.springframework.boot.loader.launch.JarLauncher 
INFO running in /deployments
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL https://172.17.27.169:8778/jolokia/

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.3)

2024-09-20T08:28:41.539Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Starting CamelApplication v1.0-SNAPSHOT using Java 21.0.3 with PID 1 (/deployments/BOOT-INF/classes started by 1000830000 in /deployments)
2024-09-20T08:28:41.544Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : No active profile set, falling back to 1 default profile: "default"
2024-09-20T08:28:44.120Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2024-09-20T08:28:44.139Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-09-20T08:28:44.139Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.28]
2024-09-20T08:28:44.213Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-09-20T08:28:44.215Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2575 ms
2024-09-20T08:28:46.885Z  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint beneath base path '/actuator'
2024-09-20T08:28:47.024Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
2024-09-20T08:28:49.057Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) is starting
2024-09-20T08:28:49.811Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Routes startup (total:1 started:1 kamelets:1)
2024-09-20T08:28:49.817Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   :     Started route1 (kamelet://postgresql-source)
2024-09-20T08:28:49.818Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) started in 754ms (build:0ms init:0ms start:754ms)
2024-09-20T08:28:49.826Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Started CamelApplication in 9.009 seconds (process running for 10.094)
2024-09-20T08:28:50.966Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-09-20T08:28:50.970Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
```

## Auto refresh of the secret and modification

To show how to refresh works we'll need to change the password for postgresadmin user on our Database.

First run the following command:

```
oc rsh $(oc get pods -l postgres-operator.crunchydata.com/role=master -o name)
```

Now you need to change the password inside the container

```
sh-4.4$ psql -U postgres -c "ALTER USER postgresadmin PASSWORD 'masteradmin12345*';"
```

To make this effective you'll need to kill the postgres instance pod.

At the same time modify the secret stored into Google Secret Manager by editing the password field with 'masteradmin12345*' in the Google console or by adding a new version of secrets through Gcloud CLI.

Now get back to the log and you should see the following entries:

```
2024-09-20T08:38:57.637Z  INFO 1 --- [          Gax-5] o.a.c.c.g.s.m.v.PubsubReloadTriggerTask  : Update for GCP secret: projects/893235743432/secrets/authsecdbref detected, triggering CamelContext reload
2024-09-20T08:38:57.638Z  INFO 1 --- [          Gax-5] o.a.c.s.DefaultContextReloadStrategy     : Reloading CamelContext (camel-1) triggered by: org.apache.camel.component.google.secret.manager.vault.PubsubReloadTriggerTask$FilteringEventMessageReceiver@5308829d
2024-09-20T08:39:10.840Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-09-20T08:39:10.844Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}


```
