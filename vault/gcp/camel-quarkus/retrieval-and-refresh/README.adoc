## GCP Secrets Manager Vault with refresh on OCP

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

Once the process complete

```
./mvnw install -Dquarkus.openshift.deploy=true
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs camel-gcp-vault-xxx
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -XX:MaxRAMPercentage=80.0 -XX:+UseParallelGC -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp "." -jar /deployments/quarkus-run.jar 
INFO running in /deployments
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2024-08-02 08:32:58,555 INFO  [org.apa.cam.qua.cor.CamelBootstrapRecorder] (main) Bootstrap runtime: org.apache.camel.quarkus.main.CamelMainRuntime
2024-08-02 08:32:58,558 INFO  [org.apa.cam.mai.MainSupport] (main) Apache Camel (Main) 4.6.0 is starting
2024-08-02 08:32:59,304 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Auto-configuration summary
2024-08-02 08:32:59,304 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.routesIncludePattern=camel/sql-to-log.camel.yaml
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.contextReloadEnabled=true
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.subscriptionName=xxxx
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.secrets=authsecdbref
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.useDefaultInstance=true
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.refreshPeriod=60000
2024-08-02 08:32:59,305 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.refreshEnabled=true
2024-08-02 08:32:59,306 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.gcp.projectId=xxxx
2024-08-02 08:33:00,250 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) is starting
2024-08-02 08:33:00,274 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Property-placeholders summary
2024-08-02 08:33:00,274 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] query=select * from test;
2024-08-02 08:33:00,274 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] dsBean=dsBean-1
2024-08-02 08:33:00,276 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] delay=5000
2024-08-02 08:33:00,277 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] password=xxxxxx
2024-08-02 08:33:00,277 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] serverName=postgres-primary.camel-gcp-sql.svc
2024-08-02 08:33:00,277 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] databaseName=test
2024-08-02 08:33:00,277 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] username=xxxxxx
2024-08-02 08:33:00,279 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Routes startup (total:1 started:1 kamelets:1)
2024-08-02 08:33:00,279 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main)     Started route1 (kamelet://postgresql-source)
2024-08-02 08:33:00,279 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.6.0 (camel-1) started in 28ms (build:0ms init:0ms start:28ms)
2024-08-02 08:33:00,393 INFO  [io.quarkus] (main) camel-gcp-vault 1.0-SNAPSHOT on JVM (powered by Quarkus 3.12.2) started in 4.090s. Listening on: http://0.0.0.0:8080
2024-08-02 08:33:00,393 INFO  [io.quarkus] (main) Profile prod activated. 
2024-08-02 08:33:00,393 INFO  [io.quarkus] (main) Installed features: [agroal, camel-attachments, camel-core, camel-google-secret-manager, camel-jackson, camel-kamelet, camel-kubernetes, camel-log, camel-microprofile-health, camel-platform-http, camel-rest, camel-rest-openapi, camel-sql, camel-yaml-dsl, cdi, kubernetes, kubernetes-client, narayana-jta, smallrye-context-propagation, smallrye-health, vertx]
2024-08-02 08:33:01,929 INFO  [route1] (Camel (camel-1) thread #2 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-02 08:33:01,932 INFO  [route1] (Camel (camel-1) thread #2 - sql://select%20*%20from%20test;) {"data":"world"}
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

At the same time modify the secret stored into Google Secret Manager by editing the password field with 'masteradmin12345*' in the Google console or by adding a new version of secrets through Gcloud CLI.

Now get back to the log and you should see the following entries:

```
2024-08-02 08:35:50,898 INFO  [org.apa.cam.com.goo.sec.man.vau.PubsubReloadTriggerTask] (Gax-5) Update for GCP secret: projects/238276835660/secrets/authsecdbref detected, triggering CamelContext reload
2024-08-02 08:35:50,898 INFO  [org.apa.cam.sup.DefaultContextReloadStrategy] (Gax-5) Reloading CamelContext (camel-1) triggered by: org.apache.camel.component.google.secret.manager.vault.PubsubReloadTriggerTask$FilteringEventMessageReceiver@6e8aac5f
2024-08-02 08:35:52,293 INFO  [route1] (Camel (camel-1) thread #5 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-02 08:35:52,294 INFO  [route1] (Camel (camel-1) thread #5 - sql://select%20*%20from%20test;) {"data":"world"}
2024-08-02 08:35:57,307 INFO  [route1] (Camel (camel-1) thread #5 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-08-02 08:35:57,308 INFO  [route1] (Camel (camel-1) thread #5 - sql://select%20*%20from%20test;) {"data":"world"}

```
