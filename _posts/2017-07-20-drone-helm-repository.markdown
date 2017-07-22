---
layout: post
title:  "Managing Helm Releases from your CI/CD"
excerpt: "How we integrated the release management of Helm into our CI/CD pipeline and why you'd want to do that."
categories: [devops]
tags: [kubernetes, devops, aws]
author: Tuan Nguyen,Vincent De Smet
---

## Context

At Honestbee we fully adopted [Helm](https://helm.sh) since late 2016 and are always focused on providing value back to the community developing these awesome open source projects.

Early on, we distilled existing best practices into a summary [Helm Best Practices](https://gist.github.com/so0k/f927a4b60003cedd101a0911757c605a) document (which was well-received, but has since been superseded by the excellent [Upstream - Helm Best Practices](https://docs.helm.sh/chart_best_practices/#the-chart-best-practices-guide) guide). We also always try to contribute to the upstream Helm Charts for the open source projects we use such as [Locust](https://kubeapps.com/charts/stable/locust), [Portus](/articles/devops/2017-06/portus-kubernetes-registry), [Fluentd](https://github.com/fluent/fluentd-kubernetes-daemonset/pull/4) and [Cloudflare-datadog](https://github.com/kubernetes/charts/pull/1324).

Another open source project we heavily rely on is [Drone.io](http://docs.drone.io/getting-started/), which is now handling CI/CD for most of our applications. Drone allows us to fully automate our application deployments into our Kubernetes clusters. A while back we also open sourced our [Drone Kubernetes plug-in](/articles/devops/2017-01/drone-kubernetes-plugin).

This post assumes familiarity with [Kubernetes](https://kubernetes.io) and some [Helm Concepts](https://docs.helm.sh/developing_charts/#charts).

## Overview and Motivations behind our Set up

Helm's templating features to facilitate the installation of our workloads on top of Kubernetes drove our initial adoption, but Helm has a lot more to offer!

In our initial set up for our team of two, we relied on a central git repository of Helm Charts which both of us kept a local copy off. Deployment secrets were handled manually outside of this repository. As the team grew and secrets were managed manually, mistakes could easily be made while changing the configuration of releases managed by Helm. Kubernetes often stopped faulty updates as soon as the first [health checks failed](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) and Helm's release management provided us powerful roll backs. However, we saw the need to fully automate releases with Helm and the first step towards this was to stop updating Kubernetes deployments directly from Drone, ensuring Helm had full control over every new release from within our Drone CI/CD Pipelines.

As we do not want our Helm Charts to live within each project directory and prefer to have Deployment artifacts de-coupled from the project (each project packages as a container image, how this container image is deployed is managed separately and may change over time). Deployments can also cross multiple projects and make use of Helm's Chart dependency mechanisms.

For this to fully work, Helm defines the concept of a central [Chart repository](https://docs.helm.sh/developing_charts/#syncing-your-chart-repository) which is very similar to the central [Docker Registry](https://docs.docker.com/registry/) used for Container Image distribution.

A Chart repository mainly consists of static assets (Although a proposal exists to treat Charts more like Container images and [re-use the existing technology used to manage image distribution](https://coreos.com/blog/quay-application-registry-for-kubernetes.html)). Any static file host (github.io / s3 / GCS / ... ) can be used, additionally - support for [auth has recently been added](https://github.com/kubernetes/helm/issues/1038) to Helm. We just need to have a pipeline to package the charts and synchronize them to the host for every change.

Next, our Drone pipelines must have a task to pull the specified Helm Chart from the Chart Repository and install or upgrade a Project's target release (Dev / Staging / Prod) within a Kubernetes cluster. For these type of pipeline tasks, which are highly re-usable across pipelines, Drone uses the concept of a [plugin](http://docs.drone.io/plugin-overview/) making them fully declarative and easily controlled through configuration.

A powerful [Helm plugin for Drone](https://github.com/ipedrazas/drone-helm) already existed, but lacked the support to work with centralized Chart Repositories. We recently forked and added this support and are working to upstream these changes.

In the next 2 section we will detail how we:

- Create and manage our private Chart Repository
- How we use this Chart Repository within our Drone pipelines (we will also provide a public sample project).

### Creating and Managing a private Chart Repository

As we are on AWS, we use s3 to host our Chart Repository, our configuration is fully defined as code using [Terraform](https://terraform.io).

Other clouds provide similar services and the setup below may be replicated there as needed.

After the storage service has been configured, we will cover how Drone is used to synchronize our Git repositories with our Helm Repository.

![vpc-endpoint](https://image.slidesharecdn.com/amazonvpcwebinar-150527181628-lva1-app6891/95/aws-may-webinar-series-deep-dive-amazon-virtual-private-cloud-57-638.jpg?cb=1432848918)

Technically, we'll create a `VPC endpoint` (named `vpce-xxxx`), and route the traffic  from clients / specific VPC towards S3 need to be passed through this one.

#### Create vpc endpoint
Firstly, we need to create `vpc endpoint` to allow private access to bucket used as private chart repository.

```bash
aws ec2 create-vpc-endpoint --vpc-id <allowed-vpc-id> --service-name com.amazonaws.ap-southeast-1.s3 --route-table-ids rtb-11aa22bb
```

- `allowed-vpc-id`: would be the one your cluster & CI server sits on

```bash
# check if vpc endpoint was successfully created

root@localhost$ aws ec2 describe-prefix-lists

{
    "PrefixLists": [
        {
            "PrefixListName": "com.amazonaws.ap-southeast-1.s3",
            "Cidrs": [
                "1.1.1.1/22",
                "2.2.2.2/22",
                "3.3.3.3/22",
                "4.4.4.4/22"
            ],
            "PrefixListId": "pl-1fa23456"
        }
    ]
}
```

#### S3 Repository set up

To ensure the repository is only accessible internally, we [restrict s3 bucket access to a particular VPC](https://stackoverflow.com/questions/25539057/restricting-s3-bucket-access-to-a-vpc) (as Helm auth support was still in flux).

Every key component is defined in code as below and takes the following parameters as variables:

- `aws_region`: The AWS Region of choice
- `domain_name`: Internal domain name. This is a private hosted zone associated with our internal VPCs.
- `bucket_name_prefix`: Prefix for the bucket on the internal domain.
- `vpc_endpoint`:  A VPC endpoint which enables you to create a private connection between your VPC and another AWS service without requiring access over the Internet. We choose to pass the id in as a parameter to this module.

Here is the Terraform configuration for the bucket policy to restrict access to our private connection:

```hcl
data  "aws_iam_policy_document" "s3-read" {
  statement {
    sid = "Access-to-specific-VPC-only"
    effect = "Allow"
    actions = [
      "s3:GetObject"
    ]
    resources = [
      "arn:aws:s3:::${var.bucket_name_prefix}.${var.domain_name}/*"
    ]

    principals {
      type        = "AWS"
      identifiers = ["*"]
    }
    condition {
      test     = "StringEquals"
      variable = "aws:sourceVpce"

      values = [
        "${var.vpc_endpoint}"
      ]
    }
  }
}
```

The s3 Bucket itself will act as a static website:

```hcl
resource "aws_s3_bucket" "charts-bucket" {
  force_destroy = true
  bucket = "${var.bucket_name_prefix}.${var.domain_name}"
  policy = "${data.aws_iam_policy_document.s3-read.json}"

  website {
    index_document = "index.html"
  }

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["PUT", "POST"]
    allowed_origins = [
      "https://${var.bucket_name_prefix}.${var.domain_name}"
    ]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}
```

And here is the configuration for a DNS record in our private zone:

```hcl
data "aws_route53_zone" "private" {
   name         = "${var.domain_name}."
   private_zone = true
 }

resource "aws_route53_record" "www" {
  zone_id = "${data.aws_route53_zone.private.zone_id}"
  name    = "${var.bucket_name_prefix}.${var.domain_name}"
  type    = "CNAME"
  ttl     = "300"
  records = ["${var.bucket_name_prefix}.${var.domain_name}.s3-website-${var.aws_region}.amazonaws.com"]
}
```

Note: Users who want to manage the bucket through the console, should get access with a policy similar to this

```
data "aws_iam_policy_document" "charts-bucket-console-access" {
  # allow listing of all buckets
  statement {
    sid = "1"
    actions = [
        "s3:ListAllMyBuckets"
    ]
    resources = [
        "arn:aws:s3:::*"
    ]
  }
  # allow all access to charts bucket
  statement {
    sid = "2"
    actions = [
      "s3:*"
    ]
    resources = [
      "${aws_s3_bucket.charts-bucket.*.arn}",
      "${formatlist("%s/*",aws_s3_bucket.charts-bucket.*.arn)}"
    ]
  }
}
```

Additionally, the IAM credentials required for the CI/CD system to access the bucket can also be generated through Terraform, but have been omitted here for brevity.

#### Synchronizing central Git repo with s3 bucket

The Drone pipeline task will require the following secrets:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION`

We use a Base Image with `bash`, `awscli` and `helm`

```dockerfile
FROM alpine:3.6
ENV HELM_VERSION 2.5.0
RUN apk --update add py-pip bash curl tar && \
  pip install awscli
RUN curl -O https://storage.googleapis.com/kubernetes-helm/helm-v${HELM_VERSION}-linux-amd64.tar.gz && \
  tar -xvf helm-v${HELM_VERSION}-linux-amd64.tar.gz && \
  cp -rfp linux-amd64/helm /usr/local/bin/ && \
  rm -rf linux-amd64*
```

Currently, we rely on the following bash script and are yet to convert it to a re-usable Drone plugin.

```bash
#!/bin/bash

# Initialize helm client locally
helm init --client-only

# only package folders which have a `Chart.yaml` at their root
charts=`find . -type f -name 'Chart.yaml'`
for c in $charts; do
  echo "Creating package for ${c%%/Chart.yaml}..."
  helm package ${c%%/Chart.yaml}
done

# Create index.yaml
helm repo index --url http://${S3_BUCKET} --home . .

echo "Syncing index.yaml and packaged charts ..."
aws s3 sync . s3://${S3_BUCKET}/ --exclude '*' --include "*.tgz" --include "index.yaml"
```

Given the above, our Drone pipeline becomes as follows:

```yaml
pipeline:
  build:
    image: registry.example.com/helm-repo-sync
    environment:
      - S3_BUCKET=helm-repo.example.com
    commands:
      - ./package.sh
    when:
      branch: [master]

```

### Managing Helm releases with a private Chart Repository and Drone

Once a Chart repository has been configured, we can use it with helm.

To illustrate how we use a Helm Repository, lets go through the manual steps first.

The repository has to be added to our Helm configuration:

```bash
$ helm repo add honestbee-charts https://github.com/honestbee/public-charts
```

Next we can search for Charts:

```bash
$ helm search hello
NAME                            VERSION DESCRIPTION
hb-charts/hello-world           0.1.0   Hello World chart for testing
honestbee-charts/hello-world    0.1.0   Hello World chart for testing
local/hello-world               0.1.0   Hello World chart for testing
```

We need to make sure our Drone deployment is running in a VPC that can access the private repository.

Finally, to manage Helm deployments from Drone, we can use our [forked Helm Drone plugin](https://github.com/honestbee/drone-helm#helm-kubernetes-plugin-for-droneio) - until we upstream those changes.

See our [Sample project](https://github.com/honestbee/hello-drone-helm) for a simple Drone pipeline example:

```yaml
pipeline:
  build_docker_image:
    image: plugins/docker
    repo: quay.io/honestbee/hello-drone-helm
    tags:
      - "latest"
      - ${DRONE_BRANCH}-${DRONE_COMMIT_SHA:0:7}
    when:
      branch: [ master, staging ]

  helm_deploy_staging:
    image: quay.io/honestbee/drone-helm
    skip_tls_verify: true
    helm_repos: public-charts=http://tech.honestbee.com/public-charts
    chart: public-charts/hello-world
    values: image.repository=quay.io/honestbee/hello-drone-helm,image.tag=${DRONE_BRANCH}-${DRONE_COMMIT_SHA:0:7}
    release: ${DRONE_REPO_NAME}-${DRONE_BRANCH}
    prefix: STAGING
    when:
      branch:
        exclude: [ master ]
```

Full Usage documentation is [available in the repository](https://github.com/honestbee/drone-helm/blob/add-repo-support/README.md#drone-pipeline-usage)

## Summary

Having Helm manage the releases on Kubernetes gives us the ability to easily roll back to a previous release where every manifest related to the Kubernetes workload is versioned (including configmaps, secrets, ingress, ... ).

We did not cover an important aspect in this article, which is the secrets management. But we will detail these in a separate article.
