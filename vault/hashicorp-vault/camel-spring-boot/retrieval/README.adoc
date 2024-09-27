## Camel-Spring-boot with Hashicorp Vault on OCP

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

Run the following command

```
./mvnw clean -DskipTests oc:deploy
```

Once everything is complete you should be able to access the logs with the following command:

```
> oc logs sql-to-log-1-xhhxt
Starting the Java application using /opt/jboss/container/java/run/run-java.sh ...
INFO exec -a "java" java -javaagent:/usr/share/java/jolokia-jvm-agent/jolokia-jvm.jar=config=/opt/jboss/container/jolokia/etc/jolokia.properties -javaagent:/usr/share/java/prometheus-jmx-exporter/jmx_prometheus_javaagent.jar=9779:/opt/jboss/container/prometheus/etc/jmx-exporter-config.yaml -XX:MaxRAMPercentage=80.0 -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -cp ".:/deployments/*" org.springframework.boot.loader.launch.JarLauncher 
INFO running in /deployments
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL https://172.17.27.166:8778/jolokia/

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.3)

2024-09-27T12:21:13.636Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Starting CamelApplication v1.0-SNAPSHOT using Java 21.0.3 with PID 1 (/deployments/BOOT-INF/classes started by 1000860000 in /deployments)
2024-09-27T12:21:13.642Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : No active profile set, falling back to 1 default profile: "default"
2024-09-27T12:21:16.433Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2024-09-27T12:21:16.461Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-09-27T12:21:16.462Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.28]
2024-09-27T12:21:16.533Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-09-27T12:21:16.535Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2794 ms
2024-09-27T12:21:18.354Z  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint beneath base path '/actuator'
2024-09-27T12:21:18.471Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
2024-09-27T12:21:19.627Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) is starting
2024-09-27T12:21:20.646Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Routes startup (total:1 started:1 kamelets:1)
2024-09-27T12:21:20.647Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   :     Started route1 (kamelet://postgresql-source)
2024-09-27T12:21:20.647Z  INFO 1 --- [           main] o.a.c.impl.engine.AbstractCamelContext   : Apache Camel 4.8.0 (camel-1) started in 1s16ms (build:0ms init:0ms start:1s16ms)
2024-09-27T12:21:20.653Z  INFO 1 --- [           main] o.e.project.sqltolog.CamelApplication    : Started CamelApplication in 7.812 seconds (process running for 8.931)
2024-09-27T12:21:21.892Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-09-27T12:21:21.899Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
2024-09-27T12:21:26.907Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"hello"}
2024-09-27T12:21:26.908Z  INFO 1 --- [%20from%20test;] route1                                   : {"data":"world"}
```


