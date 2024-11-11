---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

Improving application performance might mean you need change from the default settings to something custom. In this tutorial, you'll update your Redis cache settings by creating a ConfigMap and then have your Kubernetes pods use the new configuration settings. 



## What you'll learn


* [Create the ConfigMap](#create-the-configmap)
* [Update the Redis configuration values](#update-the-redis-configuration-values)
* [Restart the pod to verify changes](#restart-the-pod-to-verify-changes)



## Requirements

Make sure you've reviewed how to generally [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

After that, you'll need:
* A Kubernetes cluster: 
    * With at least two, preferably three, worker nodes (not control plane hosts)
    * At version 1.14 or up    
* The kubectl command-line tool configured to communicate with your cluster

{{< note >}}
If you don't have a cluster already, create one by using [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or use one of these Kubernetes playgrounds:

* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)
{{< /note >}}


<!-- lessoncontent -->


## Create the ConfigMap

ConfigMaps are a Kubernetes mechanism that lets you add configuration data into application
{{< glossary_tooltip text="pods" term_id="pod" >}}. This makes spinning up new containers faster and more consistent. 

1. Create a ConfigMap with an empty configuration block with `kubectl create configmap` by running:

    ```shell
    kubectl create configmap <map-name> <data-source>
    ```

    where \<map-name> is the name you want to assign to the ConfigMap and \<data-source> is the
    directory, file, or literal value to draw the data from.
    The name of a ConfigMap object must be a valid [DNS subdomain name](/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

    When you are creating a ConfigMap based on a file, the key in the \<data-source> defaults to
    the basename of the file, and the value defaults to the file content.
  
    The output should look like this:

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

1. Apply the ConfigMap created above, along with a Redis pod manifest:

    ```shell
    kubectl apply -f example-redis-config.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

    Examine the contents of the Redis pod manifest and note the following:

    * A volume named `config` is created by `spec.volumes[1]`
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
  `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

    This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
    ConfigMap above as `/redis-master/redis.conf` inside the pod.

    {{% code_sample file="pods/config/redis-pod.yaml" %}}

1. Examine the created objects:

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

1. Make sure that the `redis-config` key in the `example-redis-config` ConfigMap is blank:

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

1. Check the current congfiguration by using `kubectl exec` to enter the pod and run the `redis-cli` tool:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should show the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

1. Next, check `maxmemory-policy`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Which should also yield its default value of `noeviction`:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

## Update the Redis configuration values

Now let's change the `maxmemory` to a limit of 2 mb and set the `maxmemory-policy` to `allkeys-lru`, so the keys that are used the least are the ones that get removed first.

1. Add the configuration values to the `example-redis-config` ConfigMap:

    {{% code_sample file="pods/config/example-redis-config.yaml" %}}

1. Apply the updated ConfigMap:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

1. Confirm that the ConfigMap was updated:

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

1. Get into the Redis pod again with `kubectl exec` and run the `redis-cli` tool:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check the `maxmemory` by running `CONFIG GET maxmemory` and the `maxmemory-policy` by running `CONFIG GET maxmemory-policy`. You'll see that the results are still `0` and `noeviction`.

{{< note >}}
The configuration values have not changed because the pod needs to be restarted to grab the updated values from any associated ConfigMaps.
{{< /note >}}

## Restart the pod to verify changes

After you've updated the ConfigMap, you'll need to delete the pod and then apply the changes. These two actions are essentially a restart of the pod, as Kubernetes expects all declared objects (the Redis pod in this case) and automatically recreates them, if an object is missing.

1. Run these commands to delete and recreate the pod:

    ```shell
    kubectl delete pod redis
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. Get into the Redis pod and use the redis-cli tool again:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of 2097152:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

1. Lastly, `maxmemory-policy` should also be updated:

   ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It now reflects the desired value of `allkeys-lru`:

    ```shell
    1) "maxmemory-policy"
    2) "allkeys-lru"
    ```

For good housekeeping, clean up your work by deleting the resources you created during this practice:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
