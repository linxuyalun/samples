# Autoscaling service based on queue depth

## Prerequisites

### OpenFunction

You can refer to the [Installation Guide](https://github.com/OpenFunction/OpenFunction#readme) to set up OpenFunction.

### Kafka

If you don't have access to Kafka you can use these instructions to install Kafka into the cluster:

```shell
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
kubectl create ns kafka
helm install kafka confluentinc/cp-helm-charts -n kafka \
    --set cp-schema-registry.enabled=false \
    --set cp-kafka-rest.enabled=false \
    --set cp-kafka-connect.enabled=false \
    --set dataLogDirStorageClass=default \
    --set dataDirStorageClass=default \
    --set storageClass=default
kubectl rollout status deployment.apps/kafka-cp-control-center -n kafka
kubectl rollout status deployment.apps/kafka-cp-ksql-server -n kafka
kubectl rollout status statefulset.apps/kafka-cp-kafka -n kafka
kubectl rollout status statefulset.apps/kafka-cp-zookeeper -n kafka
```

When done, also deploy Kafka client and wait until it's ready:

```shell
cat <<EOF | kubectl apply -n kafka -f -
apiVersion: v1
kind: Pod
metadata:
 name: kafka-client
spec:
 containers:
  - name: kafka-client
    image: confluentinc/cp-enterprise-kafka:5.5.0
    command:
     - sh
     - -c
     - "exec tail -f /dev/null"
EOF

kubectl wait -n kafka --for=condition=ready pod kafka-client --timeout=120s
```

## Deployment

Create the `metric` topic which we will use in this sample:

> The number of `partitions` is connected to the maximum number of replicas can be scaled.

```shell
kubectl -n kafka exec -it kafka-client -- kafka-topics \
    --zookeeper kafka-cp-zookeeper-headless:2181 \
    --topic metric \
    --create \
    --partitions 10 \
    --replication-factor 3 \
    --if-not-exists
```

Generate a secret to access your container registry, such as one on [Docker Hub](https://hub.docker.com/) or [Quay.io](https://quay.io/).
You can create this secret by editing the ``REGISTRY_SERVER``, ``REGISTRY_USER`` and ``REGISTRY_PASSWORD`` fields in following command, and then run it.

  ```bash
  REGISTRY_SERVER=https://index.docker.io/v1/ REGISTRY_USER=<your_registry_user> REGISTRY_PASSWORD=<your_registry_password>
  kubectl create secret docker-registry push-secret \
      --docker-server=$REGISTRY_SERVER \
      --docker-username=$REGISTRY_USER \
      --docker-password=$REGISTRY_PASSWORD
  ```

To configure the autoscaling demo we will deploy two functions: `subscriber` which will be used to process messages of the `metric` queue in Kafka, and the `producer`, which will be publishing messages.

Modify the ``spec.image`` field in ``producer/function-producer.yaml`` and ``subscriber/function-subscribe.yaml`` to your own container registry address:

    ```yaml
    apiVersion: core.openfunction.io/v1alpha1
    kind: Function
    metadata:
      name: autoscaling-producer
    spec:
      image: "<your registry name>/autoscaling-producer:latest"
    ```

    ```yaml
    apiVersion: core.openfunction.io/v1alpha1
    kind: Function
    metadata:
      name: autoscaling-subscriber
    spec:
      image: "<your registry name>/autoscaling-subscriber:latest"
    ```

Use the following commands to create these Functions:

```shell
kubectl apply -f producer/function-producer-sample.yaml
kubectl apply -f subscriber/function-subscriber-sample.yaml
```

Back in the initial terminal now, some 20-30 seconds after the `producer` starts, you should see the number of `subscriber` pods being adjusted by Keda based on the number of the `metric` topic.


