---
layout: post
title:  "A Drone CI plugin for updating deployments in Kubernetes"
date:   2017-01-25 14:39:00 +0800
categories: devops aws kubernetes
author: Charles Martinot
---
# Context 

As we are installing more and more of our existing services (as well as all  new services) in our [Kubernetes][k8s] cluster, we reached a point where we have started moving our core apps as well. To manage our deployments, we adopted [Helm][helm]. 

# Problem

For our core apps, which are ruby monoliths, running the full test suite takes a considerable amount of time (about 15-20 minutes in Travis). Adding a Docker build to that process would push the total build time to about 20-25 minutes, even with Image layer caching. As such, we decided to experiment with in-house building and running our Docker builds in [Drone CI][droneci].

# Existing solutions

Although a [Kubernetes plugin][drone-kubernetes] and [Helm plugin][drone-helm] exist for Drone, these plugins were not satisfying our needs for the following reasons:
- The Kubernetes plugin publishes resources based on a json manifest, but we prefer to use Kubernetes managed Deployments (available since Kubernetes 1.1 ~ 1.2).
- Same thing with the Helm plugin, it doesn't allow to update an existing Deployment and leverage the built-in rolling updates.
- The Helm plugin also requires every repository to embed its own Helm chart, and we prefer to manage charts in a centralized repository.

# Our plugin 

We decided to write our own plugin, which is very simple and basically wraps `kubectl` commands. 

To be a drone plugin, the kubectl wrapper needs to be available as a Docker container.

- [The source][drone-k8s] of the plugin is publicly available
- You may use Docker to get the image: `docker pull quay.io/honestbee/drone-kubernetes`
- To run the plugin in your Drone pipeline use the following yaml:
```
# This will update the my-container-name container in a my-deployment
# deployment with the ${DRONE_COMMIT_SHA:8} tag of my-registry/my-container
pipeline:
    deploy:
      image: quay.io/honestbee/drone-kubernetes
      deployment: my-deployment
      repo: my-registry/my-container
      container: my-container-name
      namespace: my-namespace
      tag: ${DRONE_COMMIT_SHA:8}
      when:
        branch: [ master ]
```

# TL;DR

We wrote a Drone CI plugin to update Kubernetes deployments and it's available on [github][drone-k8s]. Also available as a Docker container on [Quay.io][quay] : `docker pull quay.io/honestbee/drone-kubernetes`.

[drone-k8s]: https://github.com/honestbee/drone-kubernetes
[quay]: https://quay.io/repository/honestbee/drone-kubernetes
[k8s]: https://kubernetes.io/
[droneci]: https://github.com/drone/drone
[drone-kubernetes]: https://github.com/UKHomeOffice-attic/drone-kubernetes
[drone-helm]: https://github.com/ipedrazas/drone-helm
[helm]: https://github.com/kubernetes/helm
