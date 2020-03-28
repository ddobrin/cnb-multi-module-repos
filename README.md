# Demo for building source code with multiple modules in the repository
This demo explores building images using cloud-native buildpacks for multi-module repositories, producing single/multiple images.
It goes more in-depth, past the default behavior, on how buildpacks are used 

1. [Cloud-native buildpacks](#1)
2. [Kpack](#2)

## The demo repository

This demo leverages the code from the blue-green deployment on Kubernetes repository. The repo has the source code from multiple modules, sharing a parent POM file and producing individual images (billboard-client, message-service).

This demo can also be leveraged for building single images from a repo

The code can be found here: [Source code](https://github.com/ddobrin/bluegreen-deployments-k8s) and 

Let's start with cloning the repo. You can follow building with [buildpacks](#1) and [kpack](#2) afterwards.


## What are cloud-native buildpacks
[Buildpacks](https://https://buildpacks.io/) provide a higher-level abstraction for building apps compared to Dockerfiles.

1. Provide a balance of control that reduces the operational burden on developers and supports enterprise operators who manage apps at scale.
2. Ensure that apps meet security and compliance requirements without developer intervention.
3. Provide automated delivery of both OS-level and application-level dependency upgrades, efficiently handling day-2 app operations that are often difficult to manage with Dockerfiles.
4. Rely on compatibility guarantees to safely apply patches without rebuilding artifacts and without unintentionally changing application behavior.

## What is Kpack

[kpack](https://github.com/pivotal/kpack) is a Kubernetes native system to automatically, continuously convert your applications or build artifacts into runnable Docker images.

kpack extends Kubernetes and utilizes unprivileged kubernetes primitives to provide builds of OCI images as a platform implementation of Cloud Native Buildpacks (CNB).

kpack provides a declarative image type that builds an image and schedules image rebuilds on relevant buildpack and source changes.

<a name="1"></a>
## Building modules in a repo using [cloud-native buildpacks](https://https://buildpacks.io/)

The source code:
```shell
> git clone git@github.com:ddobrin/bluegreen-deployments-k8s.git
> cd bluegreen-deployments-k8s
```

Before starting to build one of the modules, let's analyze the syntax of the ```pack``` command with an example:

```shell
> pack build <image> <path> <builder> <--publish> <env variables>

# where:
# image --> the name of the image you wish to build 
#           (ex.:triathlonguy/message-service:blue 
#                org=triathlonguy, 
#                image name=message-service, 
#                tag=blue)
# path --> the path of the repository (ex. current path . , hosts the parent POM file)
# builder --> the name of the builder to use, if not set as default (ex. cloudfoundry/cnb:bionic)
# publish --> indicates whether the image will be published to the repository
# env variables:
#   BP_BUILT_MODULE --> the module to build (ex. message-service)
#   BP_BUILD_ARGUMENTS --> build arguments 
#       pl --> Comma-delimited list of specified reactor projects to build instead
#                of all projects. A project can be specified by [groupId]:artifactId
#                   or by its relative path
#       am --> If project list is specified, also build projects required by the list

# ex: build message-service
> pack build triathlonguy/message-service:blue --publish --path . --builder cloudfoundry/cnb:bionic --env BP_BUILT_MODULE=message-service --env BP_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl message-service -am"

ex.: build billboard-client
> pack build triathlonguy/billboard-client:blue --publish --path . --builder cloudfoundry/cnb:bionic --env BP_BUILT_MODULE=billboard-client --env BP_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl billboard-client -am"
```

Using the default builder and parameters and running the ```pack`` command from the folder of the module will lead to build errors, as the path can not be resolved for all components

The important aspects to consider here:
1. BP_BUILT_MODULE sets the module to be built
2. BP_BUILD_ARGUMENTS sets the Maven build arguments where the sub-module must be specified, to avoid building every module of the repository. It is a perfomance optimization

The build process starts with the detection phase:
```shell
bionic: Pulling from cloudfoundry/cnb
Digest: sha256:7ec3b3febc25f5726f730940f0d228e9fde6700a01e8bceb4f2fd26ba49a55dd
Status: Image is up to date for cloudfoundry/cnb:bionic
===> DETECTING
[detector] 7 of 13 buildpacks participating
...
```

... followed by the code build:
```
[INFO] ------------------------------------------------------------------------
[builder] [INFO] Reactor Summary for spring-cloud-k8s 1.0.0:
[builder] [INFO]
[builder] [INFO] parent............................................. SUCCESS [  0.013 s]
[builder] [INFO] message-service ................................... SUCCESS [  3.653 s]
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] BUILD SUCCESS
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] Total time:  4.010 s
[builder] [INFO] Finished at: 2020-03-28T14:37:00Z
[builder] [INFO] ------------------------------------------------------------------------

... and results in the building of the image:
```
[exporter] Adding layer 'config'
[exporter] *** Images (sha256:18dd8b18ede78bfa0b4404919b7d570d31b2878da7606628e78062996b9c5362):
[exporter]       index.docker.io/triathlonguy/message-service:blue
[exporter] Adding cache layer 'org.cloudfoundry.openjdk:openjdk-jdk'
[exporter] Adding cache layer 'org.cloudfoundry.buildsystem:build-system-application'
[exporter] Adding cache layer 'org.cloudfoundry.buildsystem:build-system-cache'
[exporter] Adding cache layer 'org.cloudfoundry.jvmapplication:executable-jar'
[exporter] Adding cache layer 'org.cloudfoundry.springboot:spring-boot'
Successfully built image triathlonguy/message-service:blue
```

... with the image being published in the repository:
![message-service - Blue Version Deployment](https://github.com/ddobrin/cnb-multi-module-repos/blob/master/images/dockerhub.png)  


<a name="2"></a>
## Building modules in a repo using [kpack](https://github.com/pivotal/kpack)

The source code:
```shell
> git clone git@github.com:ddobrin/bluegreen-deployments-k8s.git
> cd bluegreen-deployments-k8s
```

