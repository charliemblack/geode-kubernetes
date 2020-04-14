# Apache Geode Kubernetes

This demo/sample project is the output of learning some details about kubernetes.  It allows one to install and run Apache Geode with Kubernetes.  I have tried to make it something that easy to tinker around with K8s or Apache Geode.   So feel to fork / cut ans pasted from.

The as checked-in deployment has 3 locators and 4 cache servers.  If you would like to change the size and shape of the deployment.   Just look at the locator and server StatefulSet  and change the replica count.

I personally use a tool called `yq` to review and update any values.  Since it a new tool I thought it would be good to show how to use in this context of this project.  

It is extremely common for users to deploy Geode with more or less servers or locators.   So lets review on how to change the number server replicas for the initial deployment.

```
# Read the current state of the server replicas 
demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq r server/server-statefulset.yml spec.replicas
4
# Update the current state of the server replicas
demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq w -i server/server-statefulset.yml spec.replicas 6
# Reread the current state of the number of servers that will be initially deployed.
demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq r server/server-statefulset.yml spec.replicas
6
```

Another extremely common item is the size of the server and locator processes.   Since Geode is JAVA application we alter the size of the server with some common JAVA configuration options.   Lets take a look at the current configuration and change it from 2GB to 6GB:

```
demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq r common/env-config.yml  data.SERVER_JMV_OPTS
 -mx2G -Xms2G -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=60 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSParallelRemarkEnabled -XX:+ScavengeBeforeFullGC -XX:+CMSScavengeBeforeRemark

demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq w -i common/env-config.yml --  data.SERVER_JMV_OPTS "-mx6G -Xms6G -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=60 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSParallelRemarkEnabled -XX:+ScavengeBeforeFullGC -XX:+CMSScavengeBeforeRemark"

demo@demo-virtual-machine:~/dev/geode-kubernetes$ yq r common/env-config.yml  data.SERVER_JMV_OPTS
-mx6G -Xms6G -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=60 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSParallelRemarkEnabled -XX:+ScavengeBeforeFullGC -XX:+CMSScavengeBeforeRemark
```

If we are running Geode in production we might want something larger to handle your storage requirements.  With 4 servers and 100GB of ram I have roughly 400GB of space to utilize towards storage, query results, indexing and any other overhead one would need for their Geode data needs.  To start I normally just double the applications storage requirements - then through observation we can scale up or scale down the system to be properly sized.

Why double the storage requirements.  Geode is a distributed cloud based key/value database.  Pods, nodes, physical hardware can come and go at any given time.   So the remaining Pods have to take up the slack.   Of course depending on the scale of the deployment one may need more or less overhead - just remember to observe the system. 

I have turned on Geode's resource manager which will allow any regions with overflow to utilize disk when memory pressure starts to evict data vs running out of space.

## Java VM Options Discussion

Now one might be thinking why are there all those other options.   They are just common options one turns on with server class Java Virtual Machine (JMV).    A not so common option is `XX:+UseCMSInitiatingOccupancyOnly`.  

In the JAVA VM there are ton of heuristics for calculating when to garbage collect.    Since we are storing data in RAM - we are a little different from standard application JVMs.    `UseCMSInitiatingOccupancyOnly` is just saying only start running the CMS garbage collector when we get to certain percentage.

> NOTE: Since we are using `UseCMSInitiatingOccupancyOnly` we will note while monitoring the Java GC will not run until the `UseCMSInitiatingOccupancyOnly` has been met.

### So why not G1

  Apache Geode works fine with G1GC.   The only thing that keeps me on CMS garbage collector is I can make bets on CMS for memory management.  G1 has similar concepts however the tolerances aren't as fine as CMS.

## Some common configuration to edit

* **common/env-config.yml data.SERVER_JMV_OPTS** - This will be the server Java Virtual Machine options that the server will be running with.
* **common/env-config.yml data.LOCATOR_JMV_OPTS** - This will be the locators Java Virtual Machine options that the locator will be running with.
* **common/env-config.yml data.LOCATORS** - A list of locators.  Since we are using stateful sets its ok to to fix the locators since they will be fixed in naming.   They will have the pattern of `locator-N.locator` with the FQDN of `locator-N.locator.${NAMESPACE}.svc.cluster.local`.   For more information see [Stable Network ID](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id)
* **server/server-statefulset.yml spec.replicas** - This is number of servers that will be hosting data.
* **locator/locator-statefulset.yml spec.replicas** - This is the number of locators.   For Geode a locator is the discovery service for how clients and servers find each other.  When there is a failure detected Geode is proactive in notifying its components.
  
## Data Placement Matters Because Your Data Matters

If your app doesn't care about its data or doesn't need to be accessing data during say an update - feel free to skip this part.

Great to see you reading on... Since we are talking about application data - we care about it.   Plus we are trying to make the data highly available so if one or more physical servers get taken out our application and its data isn't taken out.  

It is recommended to run servers on separate and distinct hardware from each other.  This is if the physical host that a pod is running on fails we aren't crashing the whole distributed system and loosing live data (your data is safe on the persistent volume but its not online).   So I would recommend using a hard anti-affinity policy in production.

### Example of Soft Anti Affinity - great for development sketchy for production

```yml
affinity:
    podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
        podAffinityTerm:
            labelSelector:
            matchExpressions:
                - key: "app"
                    operator: In
                    values:
                    - server
            topologyKey: "kubernetes.io/hostname"
```

### What about zones

One might be thinking about zones over hostnames - I am personally not a fan.   Zones typically map to avability zones in AWS.  Last check the "ping" time between `us-east-1` and `us-east-2` was roughly 15ms.    So if we thought about the single thread one put performance we would be rate limited to that number as we make duplicate copies from a database perspective.  

That roughly translates to 66 writes per second per thread with 15ms ping time.   Compare that to single thread write performance within a data center of 1ms or less.   I can get 1000 writes per second per thread - 15 times better.

Inter region ping times: https://www.cloudping.co

## Example of Microk8s Deployment For Local Testing

Enable DNS if you haven't already.   Geode is a distributed system and each member has to be able to resolve each other.

```console
microk8s enable dns
```

I like the idea of putting distributed database deployments in their own namespace due to the number of pods they will ultimate deploy at scale.   The `LOCATORS` property kind of assumes Geode deployments will be in their own namespace.

```console
microk8s.kubectl create -f geode-namespace.yml
microk8s.kubectl config set-context --current --namespace=geode
```
There are couple of common items in this deployment.   The big ones are the environnement and the StorageClass for Geode.

Deploy the environnement properties we need to ensure we are launching Geode with the right amount of memory etc..

```console
microk8s.kubectl apply -f common
```

Next create all of the PersistentVolume that Geode will use.   Remember Geode is a distributed system and there will be a number of persistent volumes.  I have provided a local persistent volume configuration. 

```console
microk8s.kubectl apply -f local-pv
```

Now that we have all that Geode Needs from environmental perspective lets deploy Geode starting with the Locators

```console
microk8s.kubectl apply -f locator
```

Deploy the Servers

```console
microk8s.kubectl apply -f server
```


## Helpful commands

### Delete All the Things

 ```console
microk8s.kubectl delete statefulsets,sc,services,configmaps,services,statefulsets.apps,pods,pvc,pv --all  --namespace=geode
```

### Debug Init Script

 ```console
 k logs locator-0 -c init-config
 ```

### Log into a container

 ```console
 k exec -it locator-0 -- /bin/bash
 ```

### Run GFSH in Cluster

 ```console
 k exec -it locator-0 -- /bin/bash -c gfsh
 ```

### Nuke the whole thing and start over

```console
microk8s.reset
```
> NOTE: I have had issues with running this command.   I find it useful to run the command then restart the OS and re-run the command.
