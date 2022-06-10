# OpenMPF Kubernetes

Welcome to the Open Media Processing Framework (OpenMPF) Kubernetes Project!

## What is the OpenMPF?

OpenMPF provides a platform to perform content detection and extraction on bulk
multimedia, enabling users to analyze, search, and share information through
the extraction of objects, keywords, thumbnails, and other contextual data.

OpenMPF enables users to build configurable media processing pipelines,
enabling the rapid development and deployment of analytic algorithms and
large-scale media processing applications.

### Search and Share

Simplify large-scale media processing and enable the extraction of meaningful
content

### Open API

Apply cutting-edge algorithms such as face detection and object classification

### Flexible Architecture

Integrate into your existing environment or use OpenMPF as a standalone
application

## Overview

This repository contains files that can be used to deploy OpenMPF on a
Kubernetes cluster.

## Where Am I?

- [Parent OpenMPF Project](https://github.com/openmpf/openmpf-projects)
- [OpenMPF Core](https://github.com/openmpf/openmpf)
- Components
  * [OpenMPF Standard Components](https://github.com/openmpf/openmpf-components)
  * [OpenMPF Contributed Components](https://github.com/openmpf/openmpf-contrib-components)
- Component APIs:
  * [OpenMPF C++ Component SDK](https://github.com/openmpf/openmpf-cpp-component-sdk)
  * [OpenMPF Java Component SDK](https://github.com/openmpf/openmpf-java-component-sdk)
  * [OpenMPF Python Component SDK](https://github.com/openmpf/openmpf-python-component-sdk)
- [OpenMPF Build Tools](https://github.com/openmpf/openmpf-build-tools)
- [OpenMPF Web Site Source](https://github.com/openmpf/openmpf.github.io)
- [OpenMPF Docker](https://github.com/openmpf/openmpf-docker)
- [OpenMPF Kubernetes](https://github.com/openmpf/openmpf-kubernetes) ( **You are here** )


## Getting Started
Start by installing
[`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl) locally. You will
need to configure `kubectl` to connect to a Kubernetes cluster.
[`minikube`](https://minikube.sigs.k8s.io/) can be used to create a local
cluster for development. Running `minikube start` will automatically configure
`kubectl` to use the minikube cluster. If you need to connect to a remote
cluster, the cluster administrator needs to provide you with a
[`kubeconfig`](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
file.


## Create Your Kustomize Overlay

There are many ways of configuring OpenMPF and many ways to configure a
Kubernetes cluster. To configure OpenMPF in a way that is suitable for your
environment, we recommend creating a [Kustomize](https://kustomize.io/)
overlay. The use of Kustomize is not required. You can use the yaml files
we include in our Kustomize base as an example to create your own yaml files
or to use other deployment tools.

The sections below list things you will likely want to customize for your
environment. Given the wide variety of possible configurations of Kubernetes
clusters, we cannot predict all the customization required for your environment.
The list below are only the customizations we believe will be the most common.
Kustomize allows you to make many more customizations than just those listed
below.

The general steps for setting up your overlay are:
- [Create kustomization.yaml with initial content](#create-kustomization.yaml-with-initial-content)
- [Configure shared data volume](#configure-shared-data-volume)
- [Configure database](#configure-database)
- [Select components](#select-components)
- [Change image names and tags](#select-components)
- [Configure Ingress or LoadBalancer](#configure-loadbalancer-or-ingress-controller)
- [Add any additional objects needed](#additional-resources)
- [Deploy](#deploy)

The `openmpf-kubernetes/overlays/custom` directory is ignored by git so users
can create their custom overlay in that directory. The rest of the steps assume
the user is creating their overlay in `openmpf-kubernetes/overlays/custom`,
but it is not required. If you want to create your overlay somewhere else,
you will need to adjust the relative paths used in the instructions below.


### Create kustomization.yaml with initial content ###
Start by creating a kustomization.yaml file in
`openmpf-kubernetes/overlays/custom` with the following content:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
```

### Configure shared data volume ####
OpenMPF requires a shared data volume to be mounted in the Workflow Manager,
components, and markup containers. This volume must be accessible by all nodes
in the cluster. The base overlay includes a PersistentVolumeClaim named
`openmpf-shared-data`. The PersistentVolumeClaim uses the
[ReadWriteMany access mode](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).
Only some Kubernetes volume plugins support this access mode. The best choice
of volume will vary based on your environment. One option that will work in most
cases is NFS, but it requires a preexisting NFS server. See
`example-overlays/nfs` for an example of how NFS can be configured. The volume
can be either by provisioned statically or dynamically.

If you decide to use static provisioning, you will need to first manually
create the PersistentVolume. Then, you will need to add the name of the
PersistentVolume to the `openmpf-shared-data` PersistentVolumeClaim. You also
need to explicitly set the storageClassName to `""`. This can be done in your
kustomization.yaml file. If, for example, you named the PersistentVolume
"my-openmpf-shared-volume", you can add it like this:
```yaml
patches:
  - target:
      kind: PersistentVolumeClaim
      name: openmpf-shared-data
    patch: |-
      - op: add
        path: /spec/volumeName
        value: my-openmpf-shared-volume
      - op: add
        path: /spec/storageClassName
        value: ""
```

If you are using dynamic provisioning, and you don't want the shared data volume
to use the default storage class or there is no default, you will need to add
the name of the desired storage class to the `openmpf-shared-data`
PersistentVolumeClaim. If the name of the storage class is
"example-storage-class", you can add it to your kustomization.yaml file like
this:
```yaml
patches:
  - target:
      kind: PersistentVolumeClaim
      name: openmpf-shared-data
    patch: |-
      - op: add
        path: /spec/storageClassName
        value: example-storage-class
```

The default size of `openmpf-shared-data` is 1GiB. It can be changed by
adding a patch to your `kustomization.yaml`. For example:
```yaml
patches:
  - target:
      kind: PersistentVolumeClaim
      name: openmpf-shared-data
    patch: |-
      - op: replace
        path: /spec/resources/requests/storage
        value: 10Gi
```


### Configure database ###
OpenMPF requires access to a PostgreSQL database. This can either be an
externally managed database or it can be deployed along with the rest of
OpenMPF. To deploy PostgreSQL with the rest of OpenMPF add
`- ../../optional/db` to the resources section of your kustomization.yaml file.

If you are using an externally managed database or if you want to change the
default configuration of PostgreSQL, you can modify the `mpf-secrets`
secretGenerator in your kustomization.yaml file. For example:
```yaml
secretGenerator:
  - name: mpf-secrets
    behavior: merge
    envs:
      - secret.env
    literals:
      - POSTGRES_PASSWORD=somepassword
```
See `base/secrets.env` for available secret keys and their default values.

PostgreSQL database requires a persistent volume that is accessible from
all nodes that it can be scheduled on. If it is configured to use a volume
local to node like
"[local](https://kubernetes.io/docs/concepts/storage/volumes/#local)", then
the PostgreSQL pod must be restricted to a single node.

The default size of `openmpf-db-data` is 1GiB. It can be changed by
adding a patch to your `kustomization.yaml`. For example:
```yaml
patches:
  - target:
      kind: PersistentVolumeClaim
      name: openmpf-db-data
    patch: |-
      - op: replace
        path: /spec/resources/requests/storage
        value: 10Gi
```


### Select components ###
For each component you would like deployed with OpenMPF add an entry to the
`resources` field in your kustomization.yaml file containing the path to its
directory in `openmpf-kubernetes/optional`. For example, if you wanted
to deploy the OcvFaceDetection component you would add
`- ../../optional/ocv-face-detection`. Next, you should an entry to the
`replicas` field to specify the number of instances of the component you want.
For example, if you wanted five instance of OcvFaceDetection you would add:
```yaml
replicas:
  - name: ocv-face-detection
    count: 5
```


### Change image names and tags ###
The image names for OpenMPF images do not include a registry or a tag, so you
will need to add them in your overlay. If you are going to use the same
registry and tag for all OpenMPF images, you can use the
`configure-all-mpf-images` transformer. To use the transformer you must
create a config named `mpf-image-config` with two keys, `registry` and `tag`.
If you wanted to use `my-registry` with `my-tag`, you would add the following
to your `kustomization.yaml`
```yaml
configMapGenerator:
  - name: mpf-image-config
    literals:
      - registry=my-registry
      - tag=my-tag
    options:
      labels:
        config.kubernetes.io/local-config: "true"

transformers:
  - ../../optional/configure-all-mpf-images
```

If you do not want to use the same registry and tag, you cannot use
`configure-all-mpf-images` and must specify image and tag names for
all images individually. For example:
```yaml
images:
  - name: openmpf_workflow_manager
    newName: openmpf/openmpf_workflow_manager
    newTag: some-tag
  - name: openmpf_ocv_face_detection
    newName: my-registry/openmpf_ocv_face_detection
    newTag: some-other-tag
  - name: openmpf_activemq
    newName: openmpf/openmpf_activemq
    newTag: some-tag
```

To get the full list of images used by your overlay you can run:
```bash
kubectl kustomize . | grep image:
```
in your overlay directory.


### Configure LoadBalancer or Ingress Controller ###
Depending on your environment you will need either configure a LoadBalancer or
an ingress controller to access WorkflowManager from outside the cluster.
You may also want to configure the ActiveMQ web UI to accessible from outside
the cluster.

If your cluster has a configured LoadBalancer, you can set the service type
of Workflow Manager and the ActiveMQ web UI to `LoadBalancer`. For example:
```yaml
patches:
  - target:
      kind: Service
      name: workflow-manager
    patch: |-
      - op: add
        path: /spec/type
        value: LoadBalancer
  - target:
      kind: Service
      name: activemq-web-ui
    patch: |-
      - op: add
        path: /spec/type
        value: LoadBalancer
```

If your cluster is configured to use an ingress controller, you should create
a yaml file in your overlay directory to set up the ingress. You will also need
to add the name of that file to your `kustomization.yaml` file in the
`resources` list. See `examples-overlays/nginx-ingress-https` for an example of
configuring an ingress for WorkflowManager.
`examples-overlays/nginx-ingress-https` also shows how you can set up the
ingress controller to use HTTPS. `example-overlays/tomcat-https` show how
you can configure HTTPS without an ingress controller.


### Additional Resources ####
The yaml files for any additional resources you require for your configuration,
can be added to your overlay directory. The file names must be added to the
`resources` list in your `kustomization.yaml` file.


Deploy
==================
To preview the full configuration with all the patches applied, you can run:
```bash
kubectl kustomize /path/to/overlay/dir`
```

When you are ready to deploy to your cluster, you can run:
```bash
kubectl apply -k /path/to/overlay/dir`
```

To stop OpenMPF and retain data you can run:
```bash
kubectl delete -k /path/to/overlay/dir -l '!persistent'
```

To stop OpenMPF without retaining data, you can run:
```bash
kubectl delete -k /path/to/overlay/dir
```



Local Deployment in Minikube
==================================
During development, it may be useful to test locally using
[minikube](https://minikube.sigs.k8s.io/).

- Start minikube: `minikube start`
- If you are using image from a registry that requires authentication or are
  stored locally, run: `minikube image load <image-name>` for each image you
  are using.
- Apply your overlay with `kubectl apply -k local`.
- Run `kubectl logs services/workflow-manager -f` to follow Workflow Manager's
  logs to see when it is done starting up.
- Run `kubectl port-forward service/workflow-manager 8080:8080`
- In a browser, go to `http://localhost:8080/workflow-manager`

To stop and retain data run:
```bash
kubectl delete -k local -l '!persistent'
```

To stop and delete all data run:
```bash
kubectl delete -k local
```
