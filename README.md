# tanzu-spring-cloud-gateway

A guided tutorial for experimenting with Tanzu Spring Cloud Gateway on your laptop using a docker desktop k8s cluster.

## Prerequisites

* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/docs/intro/install/#through-package-managers)
* [Tanzu Network Account](https://network.pivotal.io/) so you can download Spring Cloud Gateway for Kuberentes 

## MacOS 

* [docker desktop](https://www.docker.com/products/docker-desktop) with the 
  built-in k8s [MacOS k8s instruction](https://docs.docker.com/docker-for-mac/kubernetes/) 

## Windows 

* Windows subsystem for Linux since there are bash scripts that need to be run 
*  [docker desktop](https://www.docker.com/products/docker-desktop) with the
   built-in k8s [Windows k8s instruction](https://docs.docker.com/docker-for-windows/kubernetes/)
   
## Validate prerequisites

1. Validate your kubectl is pointing at the docker desktop built in k8s `kubectl cluster-info` you should see output
similar to what is below 

```text
Kubernetes control plane is running at https://kubernetes.docker.internal:6443
KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

2. check the version of helm you have installed using the command `helm version` this workshop has been tested with `version.BuildInfo{Version:"v3.5.0",GitCommit:"32c22239423b3b4ba6706d450bd044baffdcf9e6", GitTreeState:"dirty", GoVersion:"go1.15.6"}` earlier versions might not work.


## Download Spring Cloud Gateway for Kubernetes 

1. Download the [Spring Cloud Gateway for Kubernetes from Tanzu Net](https://network.pivotal.io/products/spring-cloud-gateway-for-kubernetes). This workshop
is tested with version 1.0.0 so make sure to download 1.0.x release. 

2. Extract the downloaded tar.gz file `tar zxf spring-cloud-gateway-k8s-1.0.0.tgz` you will find the files shown below.
```text
.
├── helm
│   └── spring-cloud-gateway-1.0.0.tgz
├── images
│   ├── gateway-1.0.0.tar
│   └── scg-operator-1.0.0.tar
└── scripts
    ├── install-spring-cloud-gateway.sh
    └── relocate-images.sh

3 directories, 5 files
```
3. Inspect the contents of the `images` folder. Notice it contains two container images packaged as `.tar` files. 
   the script located in `scripts/relocate-images.sh` is used to publish `gateway-1.0.0.tar` and 
   `scg-operator-1.0.0.tar` images into a private container registry accessible from the kubernetes cluster you 
   deploy the spring cloud gateway on.

4. Inspect the `helm` chart folder and notice that it contains a gzipped helm chart. This will 
   be used to install the spring cloud gateway operator. 
   
5. Inspect the `scripts` folder notice that it contains two shell scripts, one for publishing the private 
   container images to a registry, and the second scripts runs a helm to install the operator. 

## Load the SCGW container images into docker 

In a production deployment we would run the `scripts/relocate-images.sh` to publish the SCGW container images 
to a private container registry accessible from the k8s cluster that will be running SCGW. In this workshop
we are using Docker Desktop integrated Kubernetes, so we only need to load the container images into the local
docker desktop. 

1.  execute the command `docker load --input images/scg-operator-1.0.0.tar` to load the spring cloud gateway operator
container image into docker desktop, you should get output similar to the one below.

````text
 2021-03-04 14:05:38 ⌚  asaikali-a01 in ~/Downloads/spring-cloud-gateway-k8s-1.0.0
docker load --input images/scg-operator-1.0.0.tar 
c95d2191d777: Loading layer [==================================================>]  65.62MB/65.62MB
27502392e386: Loading layer [==================================================>]  15.87kB/15.87kB
9f10818f1f96: Loading layer [==================================================>]  3.072kB/3.072kB
3531017a5b38: Loading layer [==================================================>]  3.072kB/3.072kB
f2891f6d1a76: Loading layer [==================================================>]  25.73MB/25.73MB
d56456d8b2d1: Loading layer [==================================================>]  414.2kB/414.2kB
6dd14550f862: Loading layer [==================================================>]  128.9MB/128.9MB
576e5f5696c6: Loading layer [==================================================>]  2.362MB/2.362MB
a8b1ccd16ecc: Loading layer [==================================================>]  583.7kB/583.7kB
76dc679a925b: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0
````

2. execute the command `docker load --input images/gateway-1.0.0.tar` to load the spring cloud gateway container image
into the docker desktop, you should get output similar to the one below. 

```text
2021-03-04 14:06:34 ⌚  asaikali-a01 in ~/Downloads/spring-cloud-gateway-k8s-1.0.0
docker load --input images/gateway-1.0.0.tar 
cf8abc18e2f4: Loading layer [==================================================>]  1.406MB/1.406MB
17496f83a998: Loading layer [==================================================>]   1.67MB/1.67MB
ec0381c8f321: Loading layer [==================================================>]  6.656kB/6.656kB
c220513292ef: Loading layer [==================================================>]  143.9MB/143.9MB
0b18b1f120f4: Loading layer [==================================================>]   3.05MB/3.05MB
ab39aa8fd003: Loading layer [==================================================>]   5.12kB/5.12kB
410f6d58e75f: Loading layer [==================================================>]  750.1kB/750.1kB
c2e9ddddd4ef: Loading layer [==================================================>]  56.32kB/56.32kB
fcc507beb4cc: Loading layer [==================================================>]  4.096kB/4.096kB
c5ddeafb7442: Loading layer [==================================================>]  47.94MB/47.94MB
3c14fe2f1177: Loading layer [==================================================>]  294.4kB/294.4kB
5f70bf18a086: Loading layer [==================================================>]  1.024kB/1.024kB
b9ebca432449: Loading layer [==================================================>]    171kB/171kB
576e5f5696c6: Loading layer [==================================================>]  2.362MB/2.362MB
dedb8f01988f: Loading layer [==================================================>]  651.8kB/651.8kB
1dc94a70dbaa: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0
```

3. check your local docker images `docker images` and look for spring cloud gateway images. You should output similar
to below.
   
```text
docker images 
REPOSITORY                                                  TAG        IMAGE ID       CREATED         SIZE
dev.registry.pivotal.io/spring-cloud-gateway/scg-operator   1.0.0      35abc5be0038   41 years ago    219MB
dev.registry.pivotal.io/spring-cloud-gateway/gateway        1.0.0      0c2293e5647c   41 years ago    289M
```

## Install Spring Cloud Gateway Operator

1. create a namespace to install spring cloud gateway into using the command `kubectl create namespace spring-cloud-gateway`

1. The spring cloud gateway operator is installed using helm. We will need to create a `scg-image-values.yaml` 
   file pointing at the  registry containing the SCGW container images. `scg-image-values.yaml` file is normally 
   generated when the `scripts/relocate-images.sh` is run. We did not run the script as we don't have private container
   registry to use during the workshop so will generate `scg-image-values.yaml` manually. In the `helm` folder 
   create a file called `scg-image-values.yaml` with the content below
   
```yaml
gateway:
   image: "dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0"
scg-operator:
   image: "dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0"
```

In the previous section we loaded the scgw container images to docker-desktop which makes the images avilable to 
the k8s provided by docker desktop, making our lab simpler to run on a laptop. 

4. Execute `./scripts/install-spring-cloud-gateway.sh` you should see output similar to below 
```text
 2021-03-04 14:10:49 ⌚  asaikali-a01 in ~/Downloads/spring-cloud-gateway-k8s-1.0.0
○ → ./scripts/install-spring-cloud-gateway.sh
Installing Spring Cloud Gateway...
chart tarball: spring-cloud-gateway-1.0.0.tgz
chart name: spring-cloud-gateway
Release "spring-cloud-gateway" does not exist. Installing it now.
NAME: spring-cloud-gateway
LAST DEPLOYED: Thu Mar  4 14:18:52 2021
NAMESPACE: spring-cloud-gateway
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
This chart contains the Kubernetes operator for Spring Cloud Gateway.
Install the chart spring-cloud-gateway-crds before installing this chart
Finished installing Spring Cloud Gateway
```

5. execute the command  `kubectl get all -n spring-cloud-gateway` you should see a pod running the spring cloud gateway 
operator as shown below. 
   
```text
NAME                               READY   STATUS    RESTARTS   AGE
pod/scg-operator-679f77dbb-8w9bt   1/1     Running   0          3m42s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/scg-operator   ClusterIP   10.96.32.243   <none>        80/TCP    3m42s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scg-operator   1/1     1            1           3m42s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scg-operator-679f77dbb   1         1         1       3m42s
```

6. The spring cloud gateway operator installed three kubernetes custom resources definitions. Execute the command 
   `kubectl get crds` and you will see all the created custom CRDs shown below. These CRDs will be used to configure 
   the gateway. 
   
```text
 NAME                                              CREATED AT
springcloudgatewaymappings.tanzu.vmware.com       2021-03-04T05:47:20Z
springcloudgatewayrouteconfigs.tanzu.vmware.com   2021-03-04T05:47:20Z
springcloudgateways.tanzu.vmware.com              2021-03-04T05:47:20Z
```

## Define a gateway instance 

1. Inspect the `demo/my-gateway.yml` file it contains the YAML shown below which defines 
a spring cloud gateway instance.
   
```yaml
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGateway
metadata:
  name: my-gateway
```

2. Execute the command `kubectl apply -f demo/my-gateway.yml` which will submit a request to the cluster
to deploy an instance of spring cloud gateway. 

3. execute the command `kubectl get all` you should see a pod of the spring cloud gateway running
or being launched in the cluster's default namespace as shown in the output below.
   
```text
NAME               READY   STATUS    RESTARTS   AGE
pod/my-gateway-0   1/1     Running   0          3m36s

NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes            ClusterIP   10.96.0.1       <none>        443/TCP    3h12m
service/my-gateway            ClusterIP   10.96.178.167   <none>        80/TCP     3m36s
service/my-gateway-headless   ClusterIP   None            <none>        5701/TCP   3m36s

NAME                          READY   AGE
statefulset.apps/my-gateway   1/1     3m36s
```

4. Inspect the file `demo/route-config.yml` it contains gateway configuration CRD that proxies requests
set the gateway to github. Notice that this route configuration is generic.  

```yaml
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGatewayRouteConfig
metadata:
  name: my-gateway-routes
spec:
  routes:
    - uri: https://github.com
      predicates:
        - Path=/**
      filters:
        - StripPrefix=1
```

5. run the command `kubectl apply -f demo/route-config.yml` you 

6. Inspect the file `demo/mapping.yml` notice that it points at the gateway instance we already deployed
at the configuration defined in `route-config.yml`
```yaml
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGatewayMapping
metadata:
  name: test-gateway-mapping
spec:
  gatewayRef:
    name: my-gateway
  routeConfigRef:
    name: my-gateway-routes
```

7. run the command `kubectl apply -f demo/mapping.yml` this will configure the already deployed 
   instance to pass proxy requests to github.com 
   
8. We don't have a load balancer configured with our kind cluster, so we will use port forwarding to
   test the gateway. run the command `kubectl port-forward service/my-gateway 8080:80` you will see 
   output like
```text
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

9. Using a browser go to `http://locahost:8080` you should see the github site. The request are 
   going to spring cloud gateway which is then sending them to github.com. Congrats you have 
   managed to deploy a spring cloud gateway instance using a CRD. There are many more things
   that you can do with spring cloud gateway that we will discuss in the rest of the workshop this is 
   just the start. 
   
## Delete spring cloud gateway  

1. delete the demo gateway `kubectl delete -f demo`
2. Uninstall the operator `helm uninstall spring-cloud-gateway -n spring-cloud-gateway`

## Resources

* [GA release blog post](https://tanzu.vmware.com/content/blog/vmware-spring-cloud-gateway-kubernetes-distributed-api-gateway-generally-available)
* [documentation](https://docs.pivotal.io/scg-k8s/1-0/)