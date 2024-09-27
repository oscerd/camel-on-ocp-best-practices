## Hashicorp Vault Secrets Retrieval on OCP with Camel Quarkus

In this sample you'll use the Hashicorp Vault Properties Source and run on OCP with Camel-Spring-Boot

## Preparing the project

We'll connect to the `sec-hashicorp-vault-test` project and check the installation status. To change project, open a terminal tab and type the following command:

```
oc new-project sec-hashicorp-vault-test
```

## Setting up Hashicorp Vault instance

We are going to use Helm for this installation

First of all install the helm repository:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Then update helm repository

```
helm repo update
```

and then run the installation step:

```
helm install vault hashicorp/vault     --set "global.openshift=true"     --set "server.dev.enabled=true"     --set "server.image.repository=docker.io/hashicorp/vault"     --set "injector.image.repository=docker.io/hashicorp/vault-k8s" 
NAME: vault
LAST DEPLOYED: Fri Sep 27 09:43:11 2024
NAMESPACE: hashicorp-vault-ceq
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://developer.hashicorp.com/vault/docs


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
```

Now we need to enable the kubernetes authentication

```
oc exec -it vault-0 -- /bin/sh
```

In the command line run

```
> vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```

and

```
> vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

You should get an output of

```
Success! Data written to: auth/kubernetes/config
```

Exit from the container command line and this should complete the first part of Hashicorp Vault instance configuration.

## Setting up Database

This example uses a PostgreSQL database. We want to install it on the project `sec-hashicorp-vault-test`. This will install the operator and may take a couple of minutes to install. The operator is the Crunchy Data for Kubernetes.

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

Now, we need to find out Postgres username, password and hostname and prepare the Hashicorp Vault Secret and credentials.

```
USER_NAME=$(oc get secret postgres-pguser-postgresadmin --template={{.data.user}} | base64 -d)
USER_PASSWORD=$(oc get secret postgres-pguser-postgresadmin --template={{.data.password}} | base64 -d)
HOST=$(oc get secret postgres-pguser-postgresadmin --template={{.data.host}} | base64 -d)
PASSWORD_SKIP_SPEC_CHAR=$(sed -e 's/[&\\/]/\\&/g; s/$/\\/' -e '$s/\\$//' <<<"$USER_PASSWORD")
```

Take note of username and host, while the password is 'masteradmin1234*'

Now we need to create the secret payload. You should have your Hashicorp Vault instance running:

We need to connect to the Vault instance container:

```
oc exec -it vault-0 -- /bin/sh
```

While you are inside the container command line create the secret for authenticating with the Database

```
vault kv put secret/authsecdbref username="postgresadmin" password="masteradmin1234*" host="postgres-primary.sec-hashicorp-vault-test.svc"
====== Secret Path ======
secret/data/authsecdbref

======= Metadata =======
Key                Value
---                -----
created_time       2024-09-27T07:51:10.567975168Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Exit from the container command line.

Since the helm installation was for dev purpose, the token will be 'root'. Don't forget this.

Now we need to create a service account in the OCP cluster for our purpose of reading this secret.

```
oc apply -f sa-hashicorp-vault.yaml
```

This will create a service account named sa-hashicorp-vault

Now get back to Container command line

```
oc exec -it vault-0 -- /bin/sh
```

Run the following commands:

```
vault policy write sa-hashicorp-vault - <<EOF
path "secret/data/authsecdbref" {
  capabilities = ["read"]
}
EOF
```

You should get an output of

```
Success! Uploaded policy: sa-hashicorp-vault
```

Now let's create the Kubernetes Authentication Role

```
vault write auth/kubernetes/role/sa-hashicorp-vault \
    bound_service_account_names=sa-hashicorp-vault \
    bound_service_account_namespaces=sec-hashicorp-vault-test \
    policies=sa-hashicorp-vault \
    ttl=24h
```

This should give you an output of

```
Success! Data written to: auth/kubernetes/role/sa-hashicorp-vault
```

This complete the Database setup in combination with the Hashicorp Vault instance secrets.

## Add the properties for Hashicorp Properties function

In the application.properties file add the following field:

```
camel.vault.hashicorp.host=vault.sec-hashicorp-vault-test.svc.cluster.local
camel.vault.hashicorp.port=8200
camel.vault.hashicorp.token=root
camel.vault.hashicorp.scheme=http
```

These should already be the valid values.

## Deploy to OCP

Once the process complete

```
./mvnw install -Dquarkus.openshift.deploy=true
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs sql-to-log-787dfb4477-48wr7
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -XX:MaxRAMPercentage=80.0 -XX:+UseParallelGC -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp "." -jar /deployments/quarkus-run.jar 
INFO running in /deployments
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2024-09-27 12:51:47,036 INFO  [org.apa.cam.qua.cor.CamelBootstrapRecorder] (main) Apache Camel Quarkus 3.15.0 is starting
2024-09-27 12:51:47,041 INFO  [org.apa.cam.mai.MainSupport] (main) Apache Camel (Main) 4.8.0 is starting
2024-09-27 12:51:47,178 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Auto-configuration summary
2024-09-27 12:51:47,179 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.main.routesIncludePattern = camel/sql-to-log.camel.yaml
2024-09-27 12:51:47,179 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.hashicorp.scheme = http
2024-09-27 12:51:47,179 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.hashicorp.host = vault.sec-hashicorp-vault-test.svc.cluster.local
2024-09-27 12:51:47,180 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.hashicorp.port = 8200
2024-09-27 12:51:47,180 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [MicroProfilePropertiesSource] camel.vault.hashicorp.token = xxxxxx
2024-09-27 12:51:47,926 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.8.0 (camel-1) is starting
2024-09-27 12:51:48,543 INFO  [org.apa.cam.mai.BaseMainSupport] (main) Property-placeholders summary
2024-09-27 12:51:48,543 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] query = select * from test;
2024-09-27 12:51:48,544 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] dsBean = dsBean-1
2024-09-27 12:51:48,544 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] delay = 5000
2024-09-27 12:51:48,544 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] password = xxxxxx
2024-09-27 12:51:48,545 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] serverName = postgres-primary.sec-hashicorp-vault-test.svc
2024-09-27 12:51:48,545 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] databaseName = test
2024-09-27 12:51:48,545 INFO  [org.apa.cam.mai.BaseMainSupport] (main)     [stgresql-source.kamelet.yaml] username = xxxxxx
2024-09-27 12:51:48,548 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Routes startup (total:1 started:1 kamelets:1)
2024-09-27 12:51:48,548 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main)     Started route1 (kamelet://postgresql-source)
2024-09-27 12:51:48,548 INFO  [org.apa.cam.imp.eng.AbstractCamelContext] (main) Apache Camel 4.8.0 (camel-1) started in 621ms (build:0ms init:0ms start:621ms)
2024-09-27 12:51:48,755 INFO  [io.quarkus] (main) sql-to-log 1.0-SNAPSHOT on JVM (powered by Quarkus 3.15.1) started in 3.026s. Listening on: http://0.0.0.0:8080
2024-09-27 12:51:48,757 INFO  [io.quarkus] (main) Profile prod activated. 
2024-09-27 12:51:48,757 INFO  [io.quarkus] (main) Installed features: [agroal, camel-attachments, camel-core, camel-hashicorp-vault, camel-jackson, camel-kamelet, camel-log, camel-microprofile-health, camel-platform-http, camel-rest, camel-rest-openapi, camel-sql, camel-yaml-dsl, cdi, kubernetes, narayana-jta, smallrye-context-propagation, smallrye-health, vertx]
2024-09-27 12:51:49,667 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}
2024-09-27 12:51:49,670 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"world"}
2024-09-27 12:51:54,678 INFO  [route1] (Camel (camel-1) thread #1 - sql://select%20*%20from%20test;) {"data":"hello"}


```


