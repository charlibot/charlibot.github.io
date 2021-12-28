---
title: "Atlantis on Cloud Run"
date: 2021-12-26T22:45:12+01:00
draft: true
---

[Atlantis](https://www.runatlantis.io/) is a service developers use to apply [Terraform](https://www.terraform.io/) plans from pull requests. Atlantis needs to run *somewhere* before it can start applying Terraform changes from your PRs and this post describes how you can get Atlantis running on Google's [Cloud Run](https://cloud.google.com/run).

### Terraform and Atlantis

Engineering teams with a DevOps culture will often provision their own infrastructure from the likes of Google (GCP), Amazon (AWS) and Microsoft (Azure). This can include a managed Kubernetes cluster, a topic in Pub/Sub and an Elastic MapReduce cluster amongst many other provided services. Terraform is often used to define these as code and also provision them.

Atlantis can then be used to ensure the Terraform changes are always ran from one place (instead of each developer's machine) and integrates with a team's existing git workflow. In practice, this looks like:

1. Create a PR in the Terraform repo with your requested infrastructure changes.
1. Atlantis locks the repo, plans the changes and adds this as a comment to the PR.
1. You review the plan.
1. Optionally, you can require approval from a team member.
1. You ask Atlantis to apply the changes with a comment.

The catch is, we want Atlantis to deploy infrastructure but we need Atlantis running *somewhere* before we can start using it (🐔 and 🥚).

Atlantis is essentially a HTTP server with a UI to view locks and an endpoint to receive pull request related events. Additionally, it stores state on which PRs have locks and the plans themselves. There is a [Docker image](https://hub.docker.com/r/runatlantis/atlantis/) available. Assuming we've chosen GCP as our public cloud provider, we have a few options to run such a service:

- [Compute Engine](https://cloud.google.com/compute) 
- [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)
- [Cloud Run](https://cloud.google.com/run)

**Compute Engine** can be configured to run Docker containers but integration with the data disk necessitates an ugly startup script and some firewall rules. **GKE** on the other hand would be straightforward to setup Atlantis with, especially given the provided [Helm chart](https://github.com/runatlantis/helm-charts), however, we'd need a GKE cluster which would ideally be provisioned from Atlantis. Let's see how far we get with **Cloud Run**, a managed serverless platform to run containers.

### Cloud Run

Cloud Run can be used to reliably run a single instance of a container with a HTTP server. This makes it an potential fit for Atlantis. However, we have to be wary of some of Cloud Run's runtime features as well as leveraging other GCP products to fill out the full solution.

#### Runtime

The Atlantis service does a lot of its processing in the background, after having already responded to a pull request event. Enabling [always allocated CPU](https://cloud.google.com/run/docs/configuring/cpu-allocation) means we can reliably do this processing.

Also, we only ever want 1 single instance to be receiving events and modifying state. We can set the [maximum number of instances](https://cloud.google.com/run/docs/configuring/max-instances) to 1 but Cloud Run will only bring an instance down if another is up and able to receive traffic. In practice then, we may see more than 1 instance running in certain periods. To work around this, we can lock an instance at startup and release it when an instance is shutting down:

```shell
source /gcslock.sh
lock $ATLANTIS_BUCKET

function cleanup() {
  unlock $ATLANTIS_BUCKET
}

trap 'cleanup' SIGTERM

# rest of Atlantis server startup
```

The `gcslock.sh` script can be found at [mco-gh/gcslock](https://github.com/mco-gh/gcslock/blob/master/gcslock.sh). In the `lock` function, we can rely on Google Cloud Storage's (GCS) [strong consistency guarantees](https://cloud.google.com/storage/docs/consistency) to ensure when we try to create an object in the `ATLANTIS_BUCKET`, it will only succeed if no such object exists. If it fails to get the lock, it will retry after some backoff.

Cloud Run will send a `SIGTERM` signal to the container and wait 10 seconds before terminating a container. We catch this signal and call `unlock` which deletes the lock object, allowing the next container to get the lock and start the server. 

#### Access control



#### State

1. What is Atlantis.
  1. Chicken and egg problem. Compute engine, tiny GKE. Want smallest footprint possible but not do too much hacky stuff (previous iteration generated ssl certs and mounted them in a compute engine instance, not fun) because will be started from my machine (as a developer).
  1. HTTP server. UI for viewing locks/prs and unlocking. Events endpoint for github to send PR related activity to. 
1. What is Cloud Run. Now, with always on cpu, makes it good candidate.
1. Architecture:
  - Cloud Run runs service
  - HTTPS load balancer
  - Cloud Armor+github secret protects /events endpoint
  - IAP protects UI
  - GCS to lock
  - GCS to keep state
1. Failure considerations - worst case, everything is corrupted and need to wipe gcs bucket. Lose locks but not a big deal.
1. What the terraform looks like
1. GCS fuse. Initial spark but didn't work out. BoltDB causing issues. 
1. Just use compute engine - yeah, maybe
1. Comparison with fargate?