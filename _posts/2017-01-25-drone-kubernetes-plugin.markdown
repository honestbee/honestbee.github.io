---
layout: post
title:  "A Drone CI plugin for updating deployments in Kubernetes"
date:   2017-01-25 14:39:00 +0800
categories: devops aws kubernetes
author: Charles Martinot
---
As we are installing more and more of our services (and all the new ones) in our [Kubernetes][k8s] cluster, we reached a point where we start moving some of our core apps as well. These being monolithic ruby apps, running the full test suite is a pretty long process : about 15 minutes in Travis

Adding a Docker build to that process would push it to about 20-25 minutes, even with layer caching. We decided indeed to experiment with in-house building, running our Docker builds in [Drone CI][droneci].

There are existing [Drone plugins for Kubernetes][drone-kubernetes] or [Helm][helm], that we are also using, but they were not satisfying our needs for the following reasons : 
- The Kubernetes plugin publishes artifacts based on a json manifest, however we just want to update existing deployments with a new version of the container, through rolling deploys.
- Same thing with the Helm plugin, it doesn't allow to update an existing release.
- The Helm plugin also requires every repository to embed its own Helm chart, and we don't want to manage charts that way but instead have them in a centralized repository.

We decided to write our own plugin, which is very simple and basically wrapping `kubectl` commands. As a drone plugin it needs to be available as a Docker container. 

- [Here is the source][drone-k8s]
- Get the container : `docker pull quay.io/honestbee/drone-kubernetes`
- Run it in your Drone pipeline :
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

We wrote a Drone CI plugin to update Kubernetes deployments and it's available on [github][drone-k8s]. Also available as a Docker container on [quay.io][quay] : `docker pull quay.io/honestbee/drone-kubernetes`.

[drone-k8s]: https://github.com/honestbee/drone-kubernetes
[quay]: https://quay.io/repository/honestbee/drone-kubernetes
[k8s]: https://kubernetes.io/
[droneci]: https://github.com/drone/drone
[drone-kubernetes]: https://github.com/UKHomeOffice-attic/drone-kubernetes
[helm]: https://github.com/kubernetes/helm
