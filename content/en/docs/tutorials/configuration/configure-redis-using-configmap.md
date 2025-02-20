---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This tutorial demonstrates how to configure Redis using a Kubernetes ConfigMap, building on the foundational task of [Configuring a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/). A ConfigMap allows you to inject configuration data into application pods, keeping containerized applications flexible and portable. By using a ConfigMap, you can separate configuration from container images, making it easier to adjust settings without modifying the application itself.

## What you'll learn

In this tutorial, you'll learn how to:
* Create a ConfigMap with Redis configuration values
* Create a Redis pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applied

## Requirements

* Have a Kubernetes cluster 
  * If you don't already have a cluster, you can create one by using [minikube](https://minikube.sigs.k8s.io/docs/tutorials/kubernetes_101/module1/) or use one of these Kubernetes playgrounds:
     * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
     * [Kubernetes lab: Play with Kubernetes](https://labs.play-with-k8s.com/)
  * We recommend you run this tutorial on a cluster with at least two nodes that are not designated as control plane hosts.
* Use `kubectl` 1.14 and above
  * To check the version, run `kubectl version`.
  * Make sure your `kubect1` is configured to communicate with your cluster.


<!-- lessoncontent -->


## Get started

1. Create a ConfigMap with an empty configuration block:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

2. Apply the ConfigMap created above, along with a Redis pod manifest:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

Examine the contents of the Redis pod manifest and note the following:

* A volume named `config` is created by `spec.volumes[1]`
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
  `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
* The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

As a result, the data in `data.redis-config` from the `example-redis-config` ConfigMap is exposed as `/redis-master/redis.conf` inside the pod.

{{% code_sample file="pods/config/redis-pod.yaml" %}}

3. Examine the created objects:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Recall that we left `redis-config` key in the `example-redis-config` ConfigMap blank:

```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

4. Check the pod's current configuration:

```shell
kubectl exec -it redis -- redis-cli
```

5. Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should show the default value of `0`:

```shell
1) "maxmemory"
2) "0"
```

6. Similarly, check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It should also show the default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

Now let's add some configuration values to the ConfigMap.

7. Open the `pods/config/example-redis-config.yaml` ConfigMap and add the `redis-config` values below:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

8. Apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

9. Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values we just added:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

7. Check the Redis pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied:

```shell
kubectl exec -it redis -- redis-cli
```

8. Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

Note that it remains at the default value of `0`:

```shell
1) "maxmemory"
2) "0"
```

Similarly, check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

Note that it also remains at the default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

The configuration values have not changed because we need to delete and recreate the pod, which will grab the updated values from the ConfigMap.

9. Delete and recreate the pod:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

10. Check that the configuration values have now been updated:

```shell
kubectl exec -it redis -- redis-cli
```

11. Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should now return the updated value of `2097152`:

```shell
1) "maxmemory"
2) "2097152"
```

12. Similarly,check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It should also return the updated value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

And you're done! You've successfully created a ConfigMap with configuration values, deployed a Redis pod that mounts and utilizes the ConfigMap, and verified that the configuration was applied correctly.

Make sure to clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## Next steps

* Follow the tutorial to [Update configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
