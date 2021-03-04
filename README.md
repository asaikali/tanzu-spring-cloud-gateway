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

1. Extract the downloaded tar.gz file `tar zxf spring-cloud-gateway-k8s-1.0.0.tgz` you will find the files shown below.

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

1. inspect the contents of the `images` folder. Notice it contains two container images packaged as `.tar` files. 
   the script located in `scripts/relocate-images.sh` is used to publish `gateway-1.0.0.tar` and 
   `scg-operator-1.0.0.tar` images into a private container registry accessible from the kubernetes cluster you 
   deploy the spring cloud gateway on. In this workshop we will provide you with a container registry that already has 
   the spring cloud gateway containers available.  

1. inspect the `helm` chart folder and notice that it contains a gzipped helm chart. This will 
   be used to install the spring cloud gateway operator. The script located in `scrpits/install-spring-cloud-gateway.sh` 
   execute the helm cli to install the operator.
   
## Install Spring Cloud Gateway Operator

1. create a namespace to install spring cloud gateway into using the command `kubectl create namespace spring-cloud-gateway`

1. We are using a private container registry, so we must create a kubernetes secret in the spring-cloud-gateway 
   namespace. Grab the password for the container registry from the workshop Slack channel then modify the command
   below with the password and execute it. 

```text
kubectl create secret docker-registry spring-cloud-gateway-image-pull-secret -n spring-cloud-gateway \
--docker-server=registry.harbor.demo.tanzufun.com/tanzu \
--docker-username=workshop \
--docker-password=[replace with password from workshop]
```

1. In the `helm` folder create a file called `scg-image-values.yaml` with the content below
```yaml
gateway:
  image: "registry.harbor.demo.tanzufun.com/tanzu/gateway:1.0.0"
scg-operator:
  image: "registry.harbor.demo.tanzufun.com/tanzu/scg-operator:1.0.0"
```
this file is normally generated when the `scripts/relocate-images.sh` is run but since we did not run the script we
are creating so that the helm chart can install spring cloud gateway. 

1. Execute `./scripts/install-spring-cloud-gateway.sh` you should see output similar to below 
```text
Installing Spring Cloud Gateway...
chart tarball: spring-cloud-gateway-1.0.0.tgz
chart name: spring-cloud-gateway
Release "spring-cloud-gateway" does not exist. Installing it now.
NAME: spring-cloud-gateway
LAST DEPLOYED: Thu Mar  4 00:47:20 2021
NAMESPACE: spring-cloud-gateway
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
This chart contains the Kubernetes operator for Spring Cloud Gateway.
Install the chart spring-cloud-gateway-crds before installing this chart
Finished installing Spring Cloud Gateway
```

1. execute the command  `kubectl get all -n spring-cloud-gateway` you should a pod running the spring cloud gateway 
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

1. The spring cloud gateway operator installed three kubernetes custom resources definitions. Execute the command 
   `kubectl get crds` and you will see all the created custom crds shown below. These crds will be used to configure 
   the gateway. 
   
```text
 NAME                                              CREATED AT
springcloudgatewaymappings.tanzu.vmware.com       2021-03-04T05:47:20Z
springcloudgatewayrouteconfigs.tanzu.vmware.com   2021-03-04T05:47:20Z
springcloudgateways.tanzu.vmware.com              2021-03-04T05:47:20Z
```

## Lets proxy github 

## Resources

* [GA release blog post](https://tanzu.vmware.com/content/blog/vmware-spring-cloud-gateway-kubernetes-distributed-api-gateway-generally-available)
* [documentation](https://docs.pivotal.io/scg-k8s/1-0/)