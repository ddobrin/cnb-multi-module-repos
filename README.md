# Demo for building source code with multiple modules in the repository
This demo explores building images using cloud-native buildpacks for multi-module repositories, producing single/multiple images.
It goes more in-depth, past the default behavior, on how buildpacks are used: 

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
#   BP_MAVEN_BUILT_MODULE--> the module to build (ex. message-service)
#   BP_MAVEN_BUILD_ARGUMENTS --> build arguments 
#       pl --> Comma-delimited list of specified reactor projects to build instead
#                of all projects. A project can be specified by [groupId]:artifactId
#                   or by its relative path
#       am --> If project list is specified, also build projects required by the list

# ex: build message-service 
> pack build triathlonguy/message-service:blue --publish --path . --builder cloudfoundry/cnb:bionic --env BP_MAVEN_BUILT_MODULE=message-service --env BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl message-service -am"

# with Paketo buildpacks
> pack build triathlonguy/message-service:blue --publish --path . --builder gcr.io/paketo-buildpacks/builder:base --env BP_MAVEN_BUILT_MODULE=message-service --env BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl message-service -am"

ex.: build billboard-client
> pack build triathlonguy/billboard-client:blue --publish --path . --builder cloudfoundry/cnb:bionic --env BP_MAVEN_BUILT_MODULE=billboard-client --env BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl billboard-client -am"

# with Paketo buildpacks
> pack build triathlonguy/billboard-client:blue --publish --path . --builder gcr.io/paketo-buildpacks/builder:base --env BP_MAVEN_BUILT_MODULE=billboard-client --env BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -pl billboard-client -am"
```

Using the default builder and parameters and running the ```pack``` command from the folder of the module will lead to build errors, as the path can not be resolved for all components

The important aspects to consider here:
1. BP_MAVEN_BUILT_MODULE sets the module to be built
2. BP_MAVEN_BUILD_ARGUMENTS sets the Maven build arguments where the sub-module must be specified, to avoid building every module of the repository. It is a perfomance optimization

The build process starts with the detection, analysis and image restoration from cache phases:
```shell
===> DETECTING
[detector] 6 of 15 buildpacks participating
[detector] paketo-buildpacks/bellsoft-liberica 2.5.3
[detector] paketo-buildpacks/maven             1.2.2
[detector] paketo-buildpacks/executable-jar    1.2.3
[detector] paketo-buildpacks/apache-tomcat     1.1.3
[detector] paketo-buildpacks/dist-zip          1.3.0
[detector] paketo-buildpacks/spring-boot       1.5.3
===> ANALYZING
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:jvmkill" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:link-local-dns" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:memory-calculator" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:openssl-security-provider" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:security-providers-configurer" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:class-counter" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:java-security-properties" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:jre" from app image
[analyzer] Restoring metadata for "paketo-buildpacks/bellsoft-liberica:jdk" from cache
[analyzer] Restoring metadata for "paketo-buildpacks/maven:cache" from cache
[analyzer] Restoring metadata for "paketo-buildpacks/maven:application" from cache
[analyzer] Restoring metadata for "paketo-buildpacks/executable-jar:class-path" from app image
===> RESTORING
[restorer] Restoring data for "paketo-buildpacks/bellsoft-liberica:jdk" from cache
[restorer] Restoring data for "paketo-buildpacks/maven:application" from cache
[restorer] Restoring data for "paketo-buildpacks/maven:cache" from cache
...
```

... followed by the code build:
```
[builder] 
[builder] Paketo BellSoft Liberica Buildpack 2.5.3
[builder]     Set $BPL_JVM_HEAD_ROOM to configure the headroom in memory calculation. Default 0.
[builder]     Set $BPL_JVM_LOADED_CLASS_COUNT to configure the number of loaded classes in memory calculation. Default 35% of classes.
[builder]     Set $BPL_JVM_THREAD_COUNT to configure the number of threads in memory calculation. Default 250.
[builder]     Set $BP_JVM_VERSION to configure the Java version. Default 11.*.
[builder]   BellSoft Liberica JDK 11.0.7: Reusing cached layer
[builder]   BellSoft Liberica JRE 11.0.7: Reusing cached layer
[builder]   Memory Calculator 4.0.0: Reusing cached layer
[builder]   Class Counter: Reusing cached layer
[builder]   JVMKill Agent 1.16.0: Reusing cached layer
[builder]   Link-Local DNS: Reusing cached layer
[builder]   Java Security Properties: Reusing cached layer
[builder]   Security Providers Configurer: Reusing cached layer
[builder]   OpenSSL Security Provider 1.0.2: Reusing cached layer
[builder] 
[builder] Paketo Maven Buildpack 1.2.2
[builder]     Set $BP_MAVEN_BUILD_ARGUMENTS to configure the arguments to pass to Maven. Default -Dmaven.test.skip=true package.
[builder]     Set $BP_MAVEN_BUILT_ARTIFACT to configure the built application artifact explicitly.  Supersedes $BP_MAVEN_BUILT_MODULE. Default target/*.[jw]ar.
[builder]     Set $BP_MAVEN_BUILT_MODULE to configure the module to find application artifact in
[builder]     Set $BP_MAVEN_SETTINGS to configure  the contents of a `settings.xml` file to be used at build time
[builder]     Creating cache directory /home/cnb/.m2
[builder]   Compiled Application: Reusing cached layer
[builder]   Removing source code
[builder] 
[builder] Paketo Executable JAR Buildpack 1.2.3
[builder]   Process types:
[builder]     executable-jar: java -cp "${CLASSPATH}" ${JAVA_OPTS} org.springframework.boot.loader.JarLauncher
[builder]     task:           java -cp "${CLASSPATH}" ${JAVA_OPTS} org.springframework.boot.loader.JarLauncher
[builder]     web:            java -cp "${CLASSPATH}" ${JAVA_OPTS} org.springframework.boot.loader.JarLauncher
[builder] 
[builder] Paketo Spring Boot Buildpack 1.5.3
[builder]   Image labels:
[builder]     org.opencontainers.image.title
[builder]     org.opencontainers.image.version
[builder]     org.springframework.boot.spring-configuration-metadata.json
[builder]     org.springframework.boot.version

...
[INFO] ------------------------------------------------------------------------
[builder] [INFO] Reactor Summary for spring-cloud-k8s 1.0.0:
[builder] [INFO]
[builder] [INFO] parent............................................. SUCCESS [  0.013 s]
[builder] [INFO] billboard-client................................... SUCCESS [  3.653 s]
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] BUILD SUCCESS
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] Total time:  4.010 s
[builder] [INFO] Finished at: 2020-03-28T14:37:00Z
[builder] [INFO] ------------------------------------------------------------------------
```

... and results in the building of the image:

```
===> EXPORTING
[exporter] Reusing layer 'launcher'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:class-counter'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:java-security-properties'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:jre'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:jvmkill'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:link-local-dns'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:memory-calculator'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:openssl-security-provider'
[exporter] Reusing layer 'paketo-buildpacks/bellsoft-liberica:security-providers-configurer'
[exporter] Reusing layer 'paketo-buildpacks/executable-jar:class-path'
[exporter] Reusing 1/1 app layer(s)
[exporter] Adding layer 'config'
[exporter] *** Images (2aa64a98a973):
[exporter]       index.docker.io/triathlonguy/billboard-client:blue
[exporter] Reusing cache layer 'paketo-buildpacks/bellsoft-liberica:jdk'
[exporter] Reusing cache layer 'paketo-buildpacks/maven:application'
[exporter] Reusing cache layer 'paketo-buildpacks/maven:cache'
Successfully built image triathlonguy/billboard-client:blue
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

Download the [latest kpack release](https://github.com/pivotal/kpack/releases): you should have a ```file release-<version>.yaml```. Please note that the file is available in the ```Assets``` section of the kpack release.

Deploy kpack using kubectl:
```shell
> kubectl apply -f release-<version>.yaml

# ex.: release.-0.0.8.yaml 
# https://github.com/pivotal/kpack/releases/download/v0.0.8/release-0.0.8.yaml

# Validate that kpack is running
> kubectl -n kpack get pods

NAME                                READY   STATUS    RESTARTS   AGE
kpack-controller-5756f5b65b-f6rnn   1/1     Running   0          3d
kpack-webhook-7884b8f45b-7d22r      1/1     Running   0          24h
```

1. First, we create a builder resource for Docker images in K8s: 
```yaml
# cnb-builder.yaml

apiVersion: build.pivotal.io/v1alpha1
kind: ClusterBuilder
metadata:
  name: default
spec:
  image: cloudfoundry/cnb:bionic
```

Create it by running:
```
> kubectl apply -f cnb-builder.yaml
```

2. Second, we create a Secret for DockerHub credentials; you can substitute the credentials and repository of your choice (Docker, Harbor, etc.)
```
# dockerhub-creds.yml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-creds
  annotations:
    build.pivotal.io/docker: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
stringData:
  username: <user>
  password: <pwd>
```
Create it by running:
```
> kubectl apply -f dockerhub-creds.yml
```

3. Third, we create a Secret for Github credentials:
```
# github-creds.yml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  annotations:
    build.pivotal.io/git: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: <user>
  password: <password> or <access token for 2FA enabled accounts>
```
Create it by running:
```
> kubectl apply -f github-creds.yml
```

4. Before creating an image resource for building the Docker image for the service, we need to create a service account tying together all the required credentials

```
# kpack-service-account.yml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kpack-service-account
secrets:
  - name: dockerhub-creds
  - name: github-creds
```
Create it by running:
```
> kubectl apply -f kpack-service-account.yml 
```

5. Finally, we need to create the Image resource for building the Docker image
```
# app-source-bg-kbs.yml
apiVersion: build.pivotal.io/v1alpha1
kind: Image
metadata:
  name: message-service 
spec:
  tag: triathlonguy/message-service:blue # --> Set your Docker image
  serviceAccount: kpack-service-account
  builder:
    name: default
    kind: ClusterBuilder
  cacheSize: "1Gi"
  source:
    git:
      url: https://github.com/ddobrin/bluegreen-deployments-k8s # --> set the repo and branch from which to build
      revision: master
  build:
    env:
      - name: BP_JAVA_VERSION
        value: 11.*     # --> Java 11 is used by default, you can set the Java version you require
      - name: BP_BUILT_MODULE # --> set the module to be built 
        value: message-service
      - name: BP_BUILD_ARGUMENTS # --> set the Maven build arguments
        value: "-Dmaven.test.skip=true package -pl message-service -am"

# where env variables to be set are:
#   BP_BUILT_MODULE --> the module to build (ex. message-service)
#   BP_BUILD_ARGUMENTS --> build arguments 
#       pl --> Comma-delimited list of specified reactor projects to build instead
#                of all projects. A project can be specified by [groupId]:artifactId
#                   or by its relative path
#       am --> If project list is specified, also build projects required by the list
```

Create the Image resource:
```
> kubectl apply -f app-source-bg-kbs.yml
```

In total, we have created 5 Kubernetes resources:
* builder
* source repository secret
* image repository secret
* service account
* image 

We can follow the progress of the builds:
```shell

# at the start, status is unknown for the build
> kubectl get cnbbuilds
NAME                            IMAGE   SUCCEEDED
message-service-build-1-s227c           Unknown

# the image has been created but is in unknown state
> kubectl get image
NAME              LATESTIMAGE   READY
message-service                 Unknown

# the pod is created and executes the first step
> kubectl get pods
NAME                                      READY   STATUS     RESTARTS   AGE
message-service-build-1-s227c-build-pod   0/1     Init:0/6   0          10s

# as the build progresses
> kubectl get pods
NAME                                      READY   STATUS     RESTARTS   AGE
message-service-build-1-s227c-build-pod   0/1     Init:4/6   0          89s

# when it is complete, we can observe that the pod has completed its run
> kubectl get pods
NAME                                      READY   STATUS      RESTARTS   AGE
message-service-build-1-s227c-build-pod   0/1     Completed   0          2m58s

# we can check the logs throughout the build:
> kubectl logs message-service-build-1-s227c-build-pod
Build successful

# we check whether the build was successful:
> kubectl get cnbbuilds
NAME                            IMAGE                                                                                                                  SUCCEEDED
message-service-build-1-s227c   index.docker.io/triathlonguy/message-service@sha256:dceb137ac9133d6247aa45b629c005cec9da9a61d97aff57a55b28e147f7e6e2   True

# last, we check whether the image has been created
> kubectl get image
NAME              LATESTIMAGE                                                                                                            READY
message-service   index.docker.io/triathlonguy/message-service@sha256:dceb137ac9133d6247aa45b629c005cec9da9a61d97aff57a55b28e147f7e6e2   True
```

... with the image being published in the repository:

![message-service - Blue Version Deployment](https://github.com/ddobrin/cnb-multi-module-repos/blob/master/images/dockerhub.png)  


If we decide to remove the kpack image and the continuous build of the image, we delete the image:
```
> kubectl delete image message-service 
```
... which removes the Image and associated Pod and ClusterBuilder resources.


