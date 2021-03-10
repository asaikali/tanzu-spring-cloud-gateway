# tanzu-spring-cloud-gateway

A guided tutorial for experimenting with Tanzu Spring Cloud Gateway on your laptop using a KIND cluster.

## Prerequisites

Please make sure the most recent versions of the software packages below are
installed and working on your system. 

* [docker](https://www.docker.com/products/docker-desktop)
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/docs/intro/install/#through-package-managers)
* if you are on Windows make sure you have access to Windows subsystem for linux so that you can run bash shell scripts. 

## Validate prerequisites

1. check that docker version using `docker version`. This workshop is tested with docker version with output below.

```text
Client: Docker Engine - Community
 Cloud integration: 1.0.7
 Version:           20.10.2
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        2291f61
 Built:             Mon Dec 28 16:12:42 2020
 OS/Arch:           darwin/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.2
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       8891c58
  Built:            Mon Dec 28 16:15:28 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.3
  GitCommit:        269548fa27e0089a8b8278fc4fc781d7f65a939b
 runc:
  Version:          1.0.0-rc92
  GitCommit:        ff819c7e9184c13b7c2607fe6c30ae19403a7aff
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

2. check that kind cli is installed using `kind version` this workshop is tested with `kind v0.10.0 go1.15.7 darwin/amd64` newer versions will work older versions might not. 

3. check the kubectl version installed using `kubectl version` this workshop is tested with version 1.20.2 of the cli as shown from output 

```text
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.2", GitCommit:"faecb196815e248d3ecfb03c680a4507229c2a56", GitTreeState:"clean", BuildDate:"2021-01-14T05:14:17Z", GoVersion:"go1.15.6", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"19+", GitVersion:"v1.19.7-gke.1500", GitCommit:"99f90aac4943d3d1c8d9705458c2711240fcfc2b", GitTreeState:"clean", BuildDate:"2021-02-03T09:17:28Z", GoVersion:"go1.15.5b5", Compiler:"gc", Platform:"linux/amd64"}
```

4. check the version of helm you have installed using the command `helm version` this workshop has been tested with `version.BuildInfo{Version:"v3.5.0",GitCommit:"32c22239423b3b4ba6706d450bd044baffdcf9e6", GitTreeState:"dirty", GoVersion:"go1.15.6"}` earlier versions might not work.

## Create kind cluster

1. Create a k8s 1.20.2 cluster using the command `kind create cluster --name scgw --config cluster.yml` you should see output similar to the one below 

```text
Creating cluster "scgw" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼 
 ✓ Preparing nodes 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-scgw"
You can now use your cluster with:

kubectl cluster-info --context kind-scgw

Thanks for using kind! 😊
```

2. execute the command `kubectl get nodes` you should get the output showing a 
control plane and worker node.  

```text
NAME                 STATUS   ROLES                  AGE   VERSION
scgw-control-plane   Ready    control-plane,master   70s   v1.20.2
scgw-worker          Ready    <none>                 34s   v1.20.2
```

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
   deploy the spring cloud gateway on. In this workshop we will provide you with a container registry that already has 
   the spring cloud gateway containers available.  

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
4. Tag and Push the `scg-opertor` and `gateway` images to the local instance of regsitry in order to be use from Kind K8s cluster running in `localhost:5000`


`docker tag  dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0 localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0`
`docker push localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0`
```shell
docker push localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0
The push refers to repository [localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/scg-operator]
76dc679a925b: Layer already exists 
a8b1ccd16ecc: Layer already exists 
576e5f5696c6: Layer already exists 
6dd14550f862: Layer already exists 
d56456d8b2d1: Layer already exists 
f2891f6d1a76: Layer already exists 
3531017a5b38: Layer already exists 
9f10818f1f96: Layer already exists 
27502392e386: Layer already exists 
c95d2191d777: Layer already exists 
1.0.0: digest: sha256:e6f12f5213e9272acf6c076f2f8d24c28c990d90b3fdbaed22cb0c24bb2ef504 size: 240
```
`docker tag  dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0 localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0`
`docker push localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0`

```shell
 docker push localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0

The push refers to repository [localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/gateway]
1dc94a70dbaa: Pushed 
dedb8f01988f: Pushed 
576e5f5696c6: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
5f70bf18a086: Pushed 
b9ebca432449: Pushed 
3c14fe2f1177: Pushed 
c5ddeafb7442: Pushed 
fcc507beb4cc: Pushed 
c2e9ddddd4ef: Pushed 
410f6d58e75f: Pushed 
ab39aa8fd003: Pushed 
0b18b1f120f4: Pushed 
c220513292ef: Pushed 
ec0381c8f321: Pushed 
17496f83a998: Pushed 
cf8abc18e2f4: Pushed 
d56456d8b2d1: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
f2891f6d1a76: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
3531017a5b38: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
9f10818f1f96: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
27502392e386: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
c95d2191d777: Mounted from dev.registry.pivotal.io/spring-cloud-gateway/scg-operator 
1.0.0: digest: sha256:5f48d7eb15b9e78daf1f0ea22a0b8bf09a4157964ef831d1a8cc2b5935e64315 size: 5123
```

NOTE that the private local regitry instance must be run just verify by executing this commnad `docker ps`
 ```shell
 docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                                       NAMES
67436d0c8de6   kindest/node:v1.20.2   "/usr/local/bin/entr…"   3 minutes ago    Up 3 minutes                                                scgw-worker
9b547056976d   kindest/node:v1.20.2   "/usr/local/bin/entr…"   3 minutes ago    Up 3 minutes    127.0.0.1:52566->6443/tcp                   scgw-control-plane
312c99decac5   registry:2             "/entrypoint.sh /etc…"   13 minutes ago   Up 13 minutes   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   kind-registry
 
 ```

## Install Spring Cloud Gateway Operator

1. create a namespace to install spring cloud gateway into using the command `kubectl create namespace spring-cloud-gateway`

2. We are using a private container registry, so we must create a kubernetes secret in the spring-cloud-gateway 
   namespace. Grab the password for the container registry from the workshop Slack channel then modify the command
   below with the password and execute it. 

```text
kubectl create secret docker-registry spring-cloud-gateway-image-pull-secret -n spring-cloud-gateway \
--docker-server=registry.harbor.demo.tanzufun.com/tanzu \
--docker-username=workshop \
--docker-password=[replace with password from workshop]
```
3. In the `helm` folder create a file called `scg-image-values.yaml` with the content below
```yaml
gateway:
   image: "localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0"
scg-operator:
   image: "localhost:5000/dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0"
```
this file is normally generated when the `scripts/relocate-images.sh` is run but since we did not run the script we
are creating so that the helm chart can install spring cloud gateway. 

4. Execute `./scripts/install-spring-cloud-gateway.sh` you should see output similar to below 
```text
./scripts/install-spring-cloud-gateway.sh
Installing Spring Cloud Gateway...
chart tarball: spring-cloud-gateway-1.0.0.tgz
chart name: spring-cloud-gateway
Release "spring-cloud-gateway" does not exist. Installing it now.
NAME: spring-cloud-gateway
LAST DEPLOYED: Tue Mar  9 19:02:51 2021
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
NAME                                READY   STATUS    RESTARTS   AGE
pod/scg-operator-544df5bd98-tnmdt   1/1     Running   0          6m11s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/scg-operator   ClusterIP   10.96.210.34   <none>        80/TCP    6m11s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scg-operator   1/1     1            1           6m11s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/scg-operator-544df5bd98   1         1         1       6m11s
```

6. The spring cloud gateway operator installed three kubernetes custom resources definitions. Execute the command 
   `kubectl get crds` and you will see all the created custom CRDs shown below. These CRDs will be used to configure 
   the gateway. 
   
```shell
NAME                                              CREATED AT
springcloudgatewaymappings.tanzu.vmware.com       2021-03-10T01:02:51Z
springcloudgatewayrouteconfigs.tanzu.vmware.com   2021-03-10T01:02:51Z
springcloudgateways.tanzu.vmware.com              2021-03-10T01:02:51Z
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
![image](https://user-images.githubusercontent.com/5790758/110564893-334d5580-8113-11eb-88bf-4821ab8397d8.png)


9. Using a browser go to `http://locahost:8080` you should see the github site. The request are 
   going to spring cloud gateway which is then sending them to github.com. Congrats you have 
   managed to deploy a spring cloud gateway instance using a CRD. There are many more things
   that you can do with spring cloud gateway that we will discuss in the rest of the workshop this is 
   just the start. 
   
   ![image](https://user-images.githubusercontent.com/5790758/110564724-f8e3b880-8112-11eb-90e5-9eddee5e6114.png)

   
## Delete kind cluster 

1. execute the command `kind delete clusters scgw`

## Resources
* [KinD Local Registry](https://kind.sigs.k8s.io/docs/user/local-registry/)
* [GA release blog post](https://tanzu.vmware.com/content/blog/vmware-spring-cloud-gateway-kubernetes-distributed-api-gateway-generally-available)
* [documentation](https://docs.pivotal.io/scg-k8s/1-0/)
