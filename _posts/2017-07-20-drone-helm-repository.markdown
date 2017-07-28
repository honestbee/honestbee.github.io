---
layout: post
title:  "Managing Helm Releases from your CI/CD"
excerpt: "How we integrated the release management of Helm into our CI/CD pipeline and why you'd want to do that."
categories: [devops]
tags: [kubernetes, devops, aws]
author: Tuan Nguyen,Vincent De Smet
---

### TL;DR

We use Helm and released a Drone plugin to set up CI/CD for a private Helm Repository backed by S3. PRs to [add support for other storage services](https://github.com/honestbee/drone-helm-repo/tree/master/pkg/storage) welcome!

## Context

At Honestbee we fully adopted [Helm](https://helm.sh) since late 2016 and are always looking to provide value back to the Kubernetes community.

Early on, we distilled existing best practices into a summary [Helm Best Practices](https://gist.github.com/so0k/f927a4b60003cedd101a0911757c605a) document (the new [Upstream - Helm Best Practices](https://docs.helm.sh/chart_best_practices/#the-chart-best-practices-guide) now supersede this document). We also always try to contribute to the upstream Helm Charts for the open source projects we use such as [Locust](https://kubeapps.com/charts/stable/locust), [Portus](/articles/devops/2017-06/portus-kubernetes-registry), [Fluentd](https://github.com/fluent/fluentd-kubernetes-daemonset/pull/4) and [Cloudflare-datadog](https://github.com/kubernetes/charts/pull/1324).

Another open source project we heavily rely on is [Drone.io](http://docs.drone.io/getting-started/), which is now handling CI/CD for most of our applications. Drone allows us to fully automate our application deployments into our Kubernetes clusters. A while back we also open sourced our [Drone Kubernetes plug-in](/articles/devops/2017-01/drone-kubernetes-plugin).

In this article we provide more details on how we use Helm internally as well as give an overview of a new Drone plugin we just open sourced: [honestbee/drone-helm-repo](https://github.com/honestbee/drone-helm-repo).

Familiarity with [Kubernetes](https://kubernetes.io) and [Helm Charts](https://docs.helm.sh/developing_charts/#charts) is required to follow along.

## Overview and Motivations behind our Setup

Helm's templating features to facilitate the installation of our applications on top of our Kubernetes clusters drove initial adoption of Helm at Honestbee, but Helm is also a powerfull release manager.

In our initial set up for our team of two, we relied on a central git repository of Helm Charts which both of us kept a local copy off. Deployment secrets were handled manually outside of this repository. 

As the team grew and secrets were managed manually, mistakes were easily made while upgrading Chart Releases. Although Kubernetes often stopped faulty updates as soon as the first [health checks failed](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) and Helm provided powerful roll backs, we saw the need to fully automate releases with Helm. The first step towards this was to stop updating Kubernetes deployments directly from Drone and instead ensure Helm had full control over every new release.

We choose not to have Helm Charts living within each project directory and prefer to have Deployment artifacts de-coupled from projects (each project packages as a container image and how this container image is deployed is managed separately and may change over time). Deployments can also cross multiple projects and make use of Helm's Chart dependency mechanisms.

Here's a picture of our desired workflow:

![Drone Helm Overview](/img/posts/drone_helm_repo/overview.png)

For this to fully work, Helm defines the concept of a central [Chart repository](https://docs.helm.sh/developing_charts/#syncing-your-chart-repository) which is very similar to the central [Registry](https://docs.docker.com/registry/) Docker uses for Container Image distribution.

For Helm, a Chart repository mainly consists of static assets (Although a proposal exists to treat Charts more like Container images and [re-use the existing technology used to manage image distribution](https://coreos.com/blog/quay-application-registry-for-kubernetes.html)). Any static file host (github.io / s3 / GCS / ... ) can be used, additionally - support for [auth has recently been added](https://github.com/kubernetes/helm/issues/1038) as well. We just need to have a pipeline to package the charts and synchronize them to the hosting service each time a Chart definition changes (purple pipeline above).

Next, our Drone pipelines must have a task to pull the specified Helm Chart from the Chart Repository and install or upgrade a matching release (Dev / Staging / Prod) within a Kubernetes cluster (green pipeline above). For these type of build tasks, which are highly re-usable across pipelines, Drone uses the concept of [plugins](http://docs.drone.io/plugin-overview/). Plugins make build tasks fully declarative and easily controlled through configuration (as we'll see in the snippets below).

A powerful [Helm plugin for Drone](https://github.com/ipedrazas/drone-helm) already existed, but lacked the support to work with centralized Chart Repositories. We recently [forked](https://github.com/honestbee/drone-helm) and added this support and are working to upstream these changes.

In the next 2 sections we detail how we:

- Create and manage our private Chart Repository
- Use this Chart Repository within our Drone pipelines (we also provide a public sample project: [honestbee/hello-drone-helm](https://github.com/honestbee/hello-drone-helm)).

### Creating and Managing a private Chart Repository

As we are on AWS, we use s3 to host our Chart Repository. Our configuration is defined as code using [Terraform](https://terraform.io) and relevant snippets are shared as we walk through the setup.

Other Cloud providers have similar services (GCS / Azure Blob Storage / ...) and the setup below may be replicated there as applicable.

After the storage service has been configured, we will cover how Drone is used to synchronize our Helm Charts Git repository with our Helm Repository and how our Helm Releases are upgraded from this Private Helm repository.

#### Keeping Charts Private with AWS VPCs

To keep our S3 backed Repo private, we choose to:

- Create a `vpc endpoint`, 
- Only [allow s3 bucket access via the endpoint](https://stackoverflow.com/questions/25539057/restricting-s3-bucket-access-to-a-vpc) (using `aws:SourceVPC` access policies) 

  and 
- Route traffic from clients and other VPCs towards S3 through this endpoint.

This diagram from the Amazon Deep Dive Webinar on VPCs illustrates this set up:

![vpc-endpoint](/img/posts/drone_helm_repo/vpc-endpoint-webinar.jpg)

Create the endpoint using aws cli:

```bash
$ aws ec2 create-vpc-endpoint --vpc-id <allowed-vpc-id> --service-name com.amazonaws.ap-southeast-1.s3 --route-table-ids rtb-11aa22bb
```
Where:

- `allowed-vpc-id`: a VPC used by CI / k8s servers

Confirm the vpc endpoint exists by reviewing the prefix lists created for the s3 service.

```bash
$ aws ec2 describe-prefix-lists
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

#### Setting up the S3 components for a Helm Repository

Armed with the vpc endpoint and other parameters as below, we provide Terraform snippets for the remaining key components here:

- `aws_region`: The AWS Region of choice
- `domain_name`: Internal domain name. This is a private hosted zone associated with our internal VPCs.
- `bucket_name_prefix`: Prefix for the bucket on the internal domain.
- `vpc_endpoint`:  the VPC endpoint defined above. We choose to pass the id in as a parameter to this Terraform module.

The Terraform configuration for the bucket policy restricting access to our s3 bucket is:

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

The s3 bucket itself acting as a static website:

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

A CNAME record for the bucket in our private DNS zone of the private VPCs:

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

**Note**: Users who want to manage the bucket through the console, should get access with a policy similar to this

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

Additionally, the IAM credentials required for the CI/CD system to access the bucket can also be generated through Terraform, but this has been omitted here for brevity.

#### Synchronizing the Charts git repository to s3

We wrote a Drone plugin to package, index and push Helm Charts to s3  on every change.

With this plugin, Drone will manage your chart repository given AWS access keys and a simple yaml snippet as below:

```yaml
pipeline:
  update_helm_repo:
    image: quay.io/honestbee/drone-helm-repo
    exclude: .git
    repo_url: http://helm-charts.example.com
    storage_url: s3://helm-charts.example.com
    aws_region: ap-southeast-1
    when:
      branch: [master]
```

Full usage details are [available here](https://github.com/honestbee/drone-helm-repo#usage)


Alternatively, following bash script using `awscli` and `helm` will do the same:

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

### Managing Helm releases with a private Chart Repository and Drone

Once a Chart repository has been configured, using it with Helm manually looks as follows (we highlight the drone automation after).

The repository has to be added to our Helm configuration:

```bash
$ helm repo add honestbee-charts https://github.com/honestbee/public-charts
```

We can then search for Charts:

```bash
$ helm search hello
NAME                            VERSION DESCRIPTION
hb-charts/hello-world           0.1.0   Hello World chart for testing
honestbee-charts/hello-world    0.1.0   Hello World chart for testing
local/hello-world               0.1.0   Hello World chart for testing
```

and install a release of a particular chart as follows:

```bash
$ helm upgrade honestbee-charts/hello-world --install
```

**Note**: We need to make sure our Drone deployment is running in a VPC that can access the private repository.

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

We did not cover an important aspect in this article, which is the secrets management. But we will detail these in a future article.
