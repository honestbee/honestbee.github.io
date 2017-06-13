---
layout: post
title:  "Kickstarting your private registry with Kubernetes, Helm and Portus"
excerpt: "Using Helm and Kubernetes we can quickly provision a TLS secured private registry with User Authentication managed through Portus in a single command."
categories: [devops]
tags: [kubernetes, devops, aws]
author: Vincent De Smet
---

Part of running a private Docker Registry is managing access and authentication for the Docker images produced by your CI/CD systems. Portus is an open source project which integrates with the open source Docker Registry and provides a user interface to manage this access through teams and user namespaces. Installing, configuring and linking up each component while ensuring everything remains secure and repeatable takes a significant amount of time. In this article we will look at how powerful tools such as Docker, Kubernetes and Helm significantly speed up and simplify these tasks.

## Components

Below is a brief introduction to the three major components covered in this article.

- [Kubernetes](http://kubernetes.io) is used at Honestbee to manage our web services as well as CI/CD, ChatOps & Monitoring systems.

- [Helm](https://github.com/kubernetes/helm) is a powerfull package manager for Kubernetes as we will demonstrate in this article. Helm is a crucial tool for managing application deployments within Honestbee.

- [Portus](http://port.us.org) is an open source authorization service as well as user interface for the open source Docker Registry. Once properly configured, the Docker Registry will verify user rights with Portus. Moreover, registry webhooks will keep Portus in-sync providing a dashboard visualizing activity within the registry.

## Pre-requisites

Checklist:

- Kubernetes 1.6 cluster
- Helm
- Registry S3 Bucket and credentials
- Kubernetes Ingress controller
- Kubernetes Kube-Lego

A working Kubernetes 1.6 cluster is required for the set-up discussed in this article, we will be using a Kubernetes cluster provisioned on AWS using [Kops](https://github.com/kubernetes/kops).

Provisioning a cluster with Kops roughly goes as follows:

```
export KOPS_STATE_STORE=s3://example-kops-training-clusters-state
kops create cluster c01.example.com \
    --zones ap-southeast-1a \
    --dns-zone ZM... \
    --ssh-public-key ~/.ssh/k8s-training-example.pub
kops update cluster c01.example.com --yes
```

Once the cluster is available, ensure to install the Helm cli locally and initialise Helm in the cluster.

```
brew install kubernetes-helm
helm init
```

The Docker Registry configuration discussed also requires an S3 bucket as well as credentials to store the Docker Image layers.

```
export AWS_REGION=ap-southeast-1
export S3_BUCKET=docker-registry-example-com
aws s3 mb s3://${S3_BUCKET} --region ${AWS_REGION}
aws s3api put-object --bucket ${S3_BUCKET} --key portus/
```

Additionally, an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) controller of choice needs to be installed to manage Layer 7 load balancing into the Kubernetes cluster (See details below).

Finally, Kubernetes integration with Let's Encrypt through [kube-lego](https://github.com/jetstack/kube-lego) fully automates TLS certificate management for Ingress definitions and is required to simplify the setup (See details below).

More details about setting up these pre-requisites are provided in the [Honestbee - Portus Chart](https://github.com/honestbee/public-charts/tree/portus/stable/portus) docs.

**Note**: Hosted Kubernetes solutions also exist for:

-  Google Cloud - [GKE](https://cloud.google.com/container-engine/)
-  Microsoft Azure - [ACS](https://azure.microsoft.com/en-us/services/container-service/)

Both hosted Kubernetes solutions also provide hosted Container Registries and are not the focus of this article.

## Portus Configuration and Installation

Portus stores all state in a MariaDB instance. This Statefull component can either be managed by Kubernetes using Statefull sets and Operators or provisioned through your cloud provider.

Once MariaDB is available we will provide the configuration through a custom configuration file as follows (full sample configuration file listed at end of this section):

```yaml
portus:
  ...
  secrets:
    db:
      host: "portusdb-mariadb.default.svc.cluster.local"
      catalog: "portusdb"
      username: "portus"
      password: "portuspass"
  ...
```

The domain name used for both the Docker Registry and Portus are similarly configured:

```yaml
portus:
  fqdn: portus.registry.example.com
  ...
registry:
  fqdn: docker.registry.example.com
  ...
```

The Registry may use S3 as a storage driver (which is the assumed configuration in this article). This configuration is defined as follows:

```yaml
  ...
  config:
    s3:
      region: ap-southeast-1
      bucket: docker-registry-example-com
      rootDirectory: portus/
  secrets:
    s3:
      access-key: AKIA...
      secret-key: w8...
```

In summary, the configuration file `.secrets.yml` may look similar to:

```yaml
portus:
  fqdn: portus.registry.example.com
  secrets:
    db:
      host: "portusdb-mariadb.default.svc.cluster.local"
      catalog: "portusdb"
      username: "portus"
      password: "portuspass"
registry:
  fqdn: docker.registry.example.com
  config:
    s3:
      region: ap-southeast-1
      bucket: docker-registry-example-com
      rootDirectory: portus/
  secrets:
    s3:
      access-key: AKIA...
      secret-key: w8...
```

Until the Honestbee Portus chart [has been merged to the official Kubernetes Chart repo](https://github.com/kubernetes/charts/pull/1284), it may be installed from the Honestbee Chart Repository as follows:

1. Add Honestbee Helm Chart Repository:

   ```
   $ helm repo add honestbee-charts http://tech.honestbee.com/public-charts
   "honestbee-charts" has been added to your repositories
   $ helm repo update honestbee-charts
   Hang tight while we grab the latest from your chart repositories...
   ...
   $ helm repo list
   NAME                    URL
   stable                  http://storage.googleapis.com/kubernetes-charts
   honestbee-charts        http://tech.honestbee.com/public-charts
   $ helm search portus
   NAME                    VERSION DESCRIPTION
   honestbee-charts/portus 0.1.0   Open Source Authorization Service and User Inte...
   ```

2. Install the Portus Chart with the `.secrets.yml` discussed above:

   ```
   $ helm install honestbee-charts/portus -n portus -f .secrets.yaml
   helm install honestbee-charts/portus -n portus -f .secrets.yaml
   NAME:   portus
   LAST DEPLOYED: Tue Jun 12 09:18:35 2017
   NAMESPACE: default
   STATUS: DEPLOYED

   RESOURCES:
   ...

   NOTES:

   Portus may take up to 5 minutes to start, once ready:

   Portus can be accessed via https://portus.registry.example.com
   Registry can be accessed via https://docker.registry.example.com
   ```

As will be highlighted in the deep dive section next, several events take place within the cluster and after a few minutes we should be able to:

1. Log in as Administrator:

    ![Portus Ready](/img/posts/portus_kubernetes_registry/portus-ready.png)

2. Add the registry deployed as part of the Helm chart:

    ![Add Registry](/img/posts/portus_kubernetes_registry/portus-add-registry.png)

3. Use Portus credentials with our Docker client and push images:

    ![Docker Auth and Push](/img/posts/portus_kubernetes_registry/docker-auth-push.png)

4. Manage and Monitor Registry activity in Portus:

    ![Portus Dashboard Activity](/img/posts/portus_kubernetes_registry/portus-dashboard-activity.png)

To achieve the above, several events took place within Kubernetes and a deep dive is in order.

## Kubernetes Events - Deep Dive

Once Helm has registered the desired state with Kubernetes, the respective controllers jump into action to realise the request.

1. First, the defined Ingress resources will cause the Ingress proxy to be re-configured for the requested host names.

    ![Ingress Controller Logs](/img/posts/portus_kubernetes_registry/ingress-controller-logs.png)

2. The Kube-Lego daemon will discover the requested TLS certificates through annotations on the Ingress resources and retreive them through the ACME protocol:

    ![Kube Lego Logs](/img/posts/portus_kubernetes_registry/lego-logs.png)

3. Deployments will create Pods with Volumes mounting configuration files, secrets and TLS Certificates. Pods which failed to start due to missing secrets (if any) will be re-tried and successfully mount the TLS Certificates when they become available.

    ![Deployment secrets](/img/posts/portus_kubernetes_registry/deployment-secrets.png)

4. Portus bootstrapping scripts contained within the Docker Images will initialise the provided MariaDB endpoint.

    ![Portus init](/img/posts/portus_kubernetes_registry/portus-init.png)

5. Kubernetes HealthChecks kick in to determine when Portus is ready to serve traffic and only allow traffic when Portus is ready:

    ![portus healthchecks](/img/posts/portus_kubernetes_registry/portus-healthchecks.png)

## Summary

As illustrated in this article, infrastructure managed through Kubernetes can take advantage of several self-contained building blocks that combine to provide very powerfull features. By focusing on the minimum configuration required, Helm templating greatly simplifies application deployment on server clusters.

Using the components highlighted above allows for portable and secure application deployments.



