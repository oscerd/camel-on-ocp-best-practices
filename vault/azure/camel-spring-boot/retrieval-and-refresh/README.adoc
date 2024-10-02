## Camel-Spring-boot Azure Key Vault on OCP

In this sample you'll use the Azure Key Vault Properties Source and run on OCP with Camel-Spring-Boot

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

First of all we need to create an application

```
az ad app create --display-name test-app-key-vault
```

Then we need to obtain credentials

```
az ad app credential reset --id <appId> --append --display-name 'Description: Key Vault app client' --end-date '2024-12-31'
```

This will return a result like this


```
{
  "appId": "appId",
  "password": "pwd",
  "tenant": "tenantId"
}
```

You should take note of the password and use it as clientSecret parameter in the pom.xml, together with the clientId and tenantId.

Now create the key vault

```
az keyvault create --name <vaultName> --resource-group <resourceGroup>
```

Create a service principal associated with the application Id

```
az ad sp create --id <appId>
```

At this point we need to add a role to the application with role assignment

```
az role assignment create --assignee <appId> --role "Key Vault Administrator" --scope /subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.KeyVault/vaults/<vaultName>
```

Last step is to create policy on what can be or cannot be done with the application. In this case we just want to read the secret value. So This should be enough.

```
az keyvault set-policy --name <vaultName> --spn <appId> --secret-permissions get
```

You can create a secret through Azure CLI with the following command:

```
az keyvault secret set --name authsecdbref --vault-name <vaultName> -f postgresCreds.json
```

Where the content of secret.json could be something like:

```
{
  "username": "postgresadmin",
  "password": "xxxx",
  "host": "host"
}
```

Don't forget to modify the keyVault name in the application.properties file

Now we need to setup the Eventhub/EventGrid notification for being informed about secrets updates.

First of all we'll need a Blob account and Blob container, to track Eventhub consuming activities.

```
az storage account create --name <blobAccountName> --resource-group <resourceGroup>
```

Then create a container

```
az storage container create --account-name <blobAccountName> --name <blobContainerName>
```

Then recover the access key for this purpose

```
az storage account keys list -g <resourceGroup> -n <blobAccountName>
```

Substitute the blob Account name, blob Container name and Blob Access Key into the application.properties file.

Let's now create the Eventhub side

Create the namespace first

```
az eventhubs namespace create --resource-group <resourceGroup> --name <eventhub-namespace> --location westus --sku Standard --enable-auto-inflate --maximum-throughput-units 20
```

Now create the resource

```
az eventhubs eventhub create --resource-group <resourceGroup> --namespace-name <eventhub-namespace> --name <eventhub-name> --cleanup-policy Delete --partition-count 15
```

In the Azure portal create a shared policy for the just create eventhub resource with "MANAGE" permissions and copy the connection string.

Substitute the connection string into the application.properties.

In the Azure portal, in the key vault we're using, select events and create event subscription to event grid, by selecting "event grid schema", a system topic name of your choice and the eventhub endpoint for the just created eventhub resource.

This complete the Database and secret refresh setup.

## Deploy to OCP

Run the following command

```
./mvnw clean -DskipTests oc:deploy
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs sql-to-log-1-lzzzw
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -javaagent:/usr/share/java/jolokia-jvm-agent/jolokia-jvm.jar=config=/opt/jboss/container/jolokia/etc/jolokia.properties -javaagent:/usr/share/java/prometheus-jmx-exporter/jmx_prometheus_javaagent.jar=9779:/opt/jboss/container/prometheus/etc/jmx-exporter-config.yaml -XX:MaxRAMPercentage=80.0 -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp ".:/deployments/*" org.springframework.boot.loader.launch.JarLauncher 
INFO running in /deployments
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL https://172.17.5.148:8778/jolokia/

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.3)

2024-10-02T10:10:08.877Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Starting CamelApplication v1.0-SNAPSHOT using Java 21.0.3 with PID 1 (/deployments/BOOT-INF/classes started by 1000870000 in /deployments)
2024-10-02T10:10:08.883Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : No active profile set, falling back to 1 default profile: "default"
2024-10-02T10:10:12.287Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2024-10-02T10:10:12.310Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-10-02T10:10:12.311Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.28]
2024-10-02T10:10:12.380Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-10-02T10:10:12.382Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 3322 ms
2024-10-02T10:10:14.147Z  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint beneath base path '/actuator'
2024-10-02T10:10:14.298Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
2024-10-02T10:10:14.811Z  INFO 1 --- [           main] c.azure.identity.EnvironmentCredential   : Azure Identity => EnvironmentCredential invoking ClientSecretCredential
2024-10-02T10:10:15.120Z  INFO 1 --- [           main] c.a.c.h.n.implementation.NettyUtility    : {"az.sdk.message":"The following Netty versions were found on the classpath and have a mismatch with the versions used by azure-core-http-netty. If your application runs without issue this message can be ignored, otherwise please align the Netty versions used in your application. For more information, see https://aka.ms/azsdk/java/dependency/troubleshoot.","azure-netty-version":"4.1.110.Final","azure-netty-native-version":"2.0.65.Final","classpath-netty-version-io.netty:netty-common":"4.1.112.Final","classpath-netty-version-io.netty:netty-handler":"4.1.112.Final","classpath-netty-version-io.netty:netty-handler-proxy":"4.1.112.Final","classpath-netty-version-io.netty:netty-buffer":"4.1.112.Final","classpath-netty-version-io.netty:netty-codec":"4.1.112.Final","classpath-netty-version-io.netty:netty-codec-http":"4.1.112.Final","classpath-netty-version-io.netty:netty-codec-http2":"4.1.112.Final","classpath-netty-version-io.netty:netty-transport-native-unix-common":"4.1.112.Final","classpath-netty-version-io.netty:netty-transport-native-epoll":"4.1.112.Final","classpath-netty-version-io.netty:netty-transport-native-kqueue":"4.1.112.Final","classpath-native-netty-version-io.netty:netty-tcnative-boringssl-static":"2.0.65.Final"}
2024-10-02T10:10:16.870Z  INFO 1 --- [           main] c.azure.identity.ChainedTokenCredential  : Azure Identity => Attempted credential EnvironmentCredential returns a token
2024-10-02T10:10:16.872Z  INFO 1 --- [           main] c.a.c.implementation.AccessTokenCache    : {"az.sdk.message":"Acquired a new access token."}
2024-10-02T10:10:17.710Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) is starting
2024-10-02T10:10:18.347Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Routes startup (total:1 started:1 kamelets:1)
2024-10-02T10:10:18.348Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   :     Started route1 (kamelet://postgresql-source)
2024-10-02T10:10:18.348Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) started in 637ms (build:0ms init:0ms start:637ms)
2024-10-02T10:10:18.352Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Started CamelApplication in 10.526 seconds (process running for 11.944)
2024-10-02T10:10:19.476Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-10-02T10:10:19.479Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
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

At the same time modify the secret stored into the file postgresCreds.json with the new password and run:

```
az keyvault secret set --name authsecdbref --vault-name <keyvaultName> -f postgresCreds.json
```

Now get back to the log and you should see the following entries:

```
2024-10-02T13:33:50.659Z  INFO 1 --- [ition-pump-3-18] o.a.c.c.a.k.v.EventhubsReloadTriggerTask : Update for Azure secret: authsecdbref detected, triggering CamelContext reload
2024-10-02T13:33:50.659Z  INFO 1 --- [ition-pump-3-18] o.a.c.s.DefaultContextReloadStrategy     : Reloading CamelContext (camel-1) triggered by: Azure Secrets Refresh Task
2024-10-02T13:33:52.085Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-10-02T13:33:52.085Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
```
