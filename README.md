# CRDB Kubernetes Operator Log Config

In this repo we look at how to override the default logging configuration for CockroachDB when using the Kubernetes Operator. You need a Kubernetes cluster with three worker nodes. Firstly we will deploy the Operator.

First we need to extend the Kubernetes API with the CockroachDB Custom Resource Definition.

Create the CRD
```
kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.14.0/install/crds.yaml
```

Once the API is extended we can deploy the Operator itself. An Operator is a piece of software that runs inside a pod. This software performs administration tasks typically performed by a human operator.

Install the Operator
```
kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.14.0/install/operator.yaml
```

Create a namespace to deploy CockroachDB into.
```
kubectl create ns cockroachdb
```
To override the default logging configuration a ConfigMap is used. This is deployed into the same namespace where you are going to deploy CockroachDB.
Below is an example Configuration.
```
apiVersion: v1
data:
  logging.yaml: |
    sinks:
      file-groups:
        dev:
          channels: DEV
          filter: WARNING
      fluent-servers:
        ops:
          channels: [OPS, HEALTH, SQL_SCHEMA]
          address: 127.0.0.1:5170
          net: tcp
          redact: true
        security:
          channels: [SESSIONS, USER_ADMIN, PRIVILEGES, SENSITIVE_ACCESS]
          address: 127.0.0.1:5170
          net: tcp
          auditable: true
kind: ConfigMap
metadata:
  name: logconfig
  namespace: cockroachdb
```

To deploy the ConfigMap run the command below.
```
kubectl apply -f logging.yaml -n cockroachdb
```

To see the presents of the configmap run the following command
```
kubectl get cm -n cockroachdb
```

Do deploy CockroachDB via the Operator you need a manifest that describe the configuratio you can download an example via the URL below.
```
curl -O https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.14.0/examples/example.yaml
```

To ensure that the logging config is picked up from the configmap you need to add the following setting under the `spec` section of the manifest.
```
spec:
  logConfigMap: logconfig
```

Then apply the `example.yaml`
```
kubectl apply -f example.yaml -n cockroachdb
```

If you do a `describe` on one of the CockroachDB pods you will see the new logging config applied.
```
    Command:
      /bin/bash
      -ecx
      exec /cockroach/cockroach.sh start --advertise-host=$(POD_NAME).cockroachdb.cockroachdb --certs-dir=/cockroach/cockroach-certs/ --http-port=8080 --sql-addr=:26257 --listen-addr=:26258 --log="sinks:
        file-groups:
          dev:
            channels: DEV
            filter: WARNING
        fluent-servers:
          ops:
            channels: [OPS, HEALTH, SQL_SCHEMA]
            address: 127.0.0.1:5170
            net: tcp
            redact: true
          security:
            channels: [SESSIONS, USER_ADMIN, PRIVILEGES, SENSITIVE_ACCESS]
            address: 127.0.0.1:5170
            net: tcp
            auditable: true
      " --cache $(expr $MEMORY_LIMIT_MIB / 4)MiB --max-sql-memory $(expr $MEMORY_LIMIT_MIB / 4)MiB --join=cockroachdb-0.cockroachdb.cockroachdb:26258,cockroachdb-1.cockroachdb.cockroachdb:26258,cockroachdb-2.cockroachdb.cockroachdb:26258
   ```