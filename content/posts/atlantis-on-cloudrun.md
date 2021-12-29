---
title: "Atlantis on Cloud Run"
date: 2021-12-26T22:45:12+01:00
draft: false
tags: [
    "gcp",
    "terraform",
    "cloudrun",
    "devops",
]
---

[Atlantis](https://www.runatlantis.io/) is a service developers use to apply [Terraform](https://www.terraform.io/) plans from pull requests. Atlantis needs to run *somewhere* before it can start applying Terraform changes from your PRs and this post describes how to get Atlantis running on Google's [Cloud Run](https://cloud.google.com/run).

Whilst this post is centred on getting Atlantis to run on Cloud Run, it touches on a few of Google's other products and their integrations with Cloud Run so may be of interest even if Atlantis is not.

## Terraform and Atlantis

Engineering teams with a DevOps culture will often provision their own infrastructure from the likes of Google (GCP), Amazon (AWS) and Microsoft (Azure). This can include a managed Kubernetes cluster, a topic in Pub/Sub or an Elastic MapReduce cluster amongst many other provided services. Terraform is often used to define these as code and also provision them.

Atlantis can be used to ensure the Terraform changes are always ran from one place (instead of each developer's machine) and integrates with a team's existing git workflow. This looks like:

1. Create a PR in the Terraform repo with your requested infrastructure changes.
1. Atlantis locks the repo, plans the changes and adds this as a comment to the PR.
1. You review the plan.
1. Optionally, you can require approval from a team member.
1. You ask Atlantis to apply the changes with a comment.

The catch is we want Atlantis to deploy infrastructure but we need Atlantis running *somewhere* before we can start using it (🐔 and 🥚).

Atlantis is essentially a HTTP server with a UI to view locks and an endpoint to receive pull request related events. It stores state on which PRs have locks and the plans themselves. Additionally, there is a [Docker image](https://hub.docker.com/r/runatlantis/atlantis/) available. Assuming we've chosen GCP as our public cloud provider, we have a few options to run such a service:

- [Compute Engine](https://cloud.google.com/compute) 
- [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)
- [Cloud Run](https://cloud.google.com/run)

**Compute Engine** can be configured to run Docker containers but integration with the Atlantis data disk necessitates an ugly startup script and some firewall rules. **GKE** on the other hand would be straightforward to setup Atlantis with, especially given the provided [Helm chart](https://github.com/runatlantis/helm-charts). However, we'd need a GKE cluster which would ideally be provisioned from Atlantis. Let's see how far we get with **Cloud Run**, a managed serverless platform to run containers.

## Cloud Run

Cloud Run can be used to reliably run a single instance of a container with a HTTP server. This makes it a potential fit for Atlantis. We have to be wary though of some of Cloud Run's runtime characteristics as well as leveraging other GCP products to fill out the full solution.

### Runtime

The Atlantis service does a lot of its processing in the background, after having already responded to a pull request event. Enabling [always allocated CPU](https://cloud.google.com/run/docs/configuring/cpu-allocation) means we can do this processing.

Secondly, we only ever want 1 single instance to be receiving events and modifying state. We can set the [maximum number of instances](https://cloud.google.com/run/docs/configuring/max-instances) to 1 but Cloud Run will only bring an instance down if another is up and able to receive traffic. In practice then, we may see more than 1 instance running in certain periods. To work around this, we can lock an instance at startup and release it when an instance is shutting down:

```shell
source /gcslock.sh
lock $ATLANTIS_BUCKET

function cleanup() {
  unlock $ATLANTIS_BUCKET
}

trap 'cleanup' SIGTERM

# rest of Atlantis server startup
```

The `gcslock.sh` script can be found at [mco-gh/gcslock](https://github.com/mco-gh/gcslock/blob/master/gcslock.sh). The `lock` function tries to create an object with the header `x-goog-if-generation-match:0`. We can rely on Google Cloud Storage's (GCS) [strong consistency guarantees](https://cloud.google.com/storage/docs/consistency) to ensure when we try to create this object in the `ATLANTIS_BUCKET`, it will only succeed if no such object exists. If it fails to create the object, and thus get the lock, it will continually retry with an increasing backoff between retries.

When shutting down, Cloud Run will send a `SIGTERM` signal to the container and wait 10 seconds before terminating a container. We catch this signal and call `unlock` which deletes the lock object, allowing the next container to get the lock and start the server. 

### Access control

When a Cloud Run service is deployed, GCP provides a default URL that can be used to call the service. Also, by default, services must be called by authorized users and service accounts with their ID token in the request's `Authorization` header.

At least with Github, there are no options to send pull request events with an ID token. A [secret token](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks) is the only option. Therefore, we must enable [public access](https://cloud.google.com/run/docs/authenticating/public) to our Cloud Run Atlantis service.

Now, we can ensure the `/events` endpoint is secure with a secret token. However, we have nothing to protect us against nefarious actors discarding plans and unlocking PRs from the UI.

At this point, we have a couple of options:

1. Create a backend and associated [load balancer](https://cloud.google.com/load-balancing) with [Cloud Armor](https://cloud.google.com/armor) rules to limit the IP addresses that can access the service to Github and the workplace.
1. Create two backends, one for `/events` which can be open (or apply the Cloud Armor rules from above) and another backend matching everything else to be protected by [Identity Aware Proxy](https://cloud.google.com/iap).

In both cases we can use a custom domain and managed SSL certificate. Let's focus on the second approach and see what the Terraform code for the load balancer module looks like:

```terraform
module "atlantis-lb-https" {
  # We use the serverless negs module from https://github.com/terraform-google-modules/terraform-google-lb-http/tree/master/modules/serverless_negs
  source  = "GoogleCloudPlatform/lb-http/google//modules/serverless_negs"
  version = "~> 5.1"
  name    = "atlantis"
  project = var.project_id

  # Enable SSL support with a managed certificate and HTTPS redirect
  ssl                             = true
  managed_ssl_certificate_domains = [var.domain]
  https_redirect                  = true

  # This map is where we define which backend gets which requests based on the path. We will see the resource definition later.
  url_map        = google_compute_url_map.atlantis_url_map.self_link
  create_url_map = false

  backends = {
    # Default backend for the UI with IAP enabled
    default = {
      description = null
      groups = [
        {
          group = google_compute_region_network_endpoint_group.serverless_neg.id
        }
      ]
      enable_cdn              = false
      security_policy         = null
      custom_request_headers  = null
      custom_response_headers = null

      iap_config = {
        enable               = true
        oauth2_client_id     = var.oauth2_client_id
        oauth2_client_secret = var.oauth2_client_secret
      }
      log_config = {
        enable      = false
        sample_rate = null
      }
    }

    # Backend for /events with IAP disabled but with the security policy set to github_only_policy
    events = {
      description = "Backend for Atlantis' /events endpoint"
      groups = [
        {
          group = google_compute_region_network_endpoint_group.serverless_neg.id
        }
      ]
      enable_cdn              = false
      security_policy         = google_compute_security_policy.github_only_policy.name
      custom_request_headers  = null
      custom_response_headers = null

      iap_config = {
        enable               = false
        oauth2_client_id     = ""
        oauth2_client_secret = ""
      }
      log_config = {
        enable      = false
        sample_rate = null
      }
    }
  }
}
```

The Terraform code above refers to some resources that are defined below. These may need some tweaking for your own setup but the general idea is there.

```terraform
resource "google_compute_region_network_endpoint_group" "atlantis_serverless_neg" {
  provider              = google-beta
  name                  = "atlantis-serverless-neg"
  network_endpoint_type = "SERVERLESS"
  region                = var.region
  cloud_run {
    # The name of the Cloud Run service. This can also be defined in Terraform and referred to here
    service = "atlantis"
  }
}

resource "google_compute_security_policy" "github_only_policy" {
  name = "github-only-policy"

  rule {
    action   = "allow"
    priority = "1000"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = [
          "192.30.252.0/22",
          "185.199.108.0/22",
          "140.82.112.0/20",
          "143.55.64.0/20",
          "2a0a:a440::/29",
          "2606:50c0::/32",
        ]
      }
    }
    description = "Allow access from Github's webhook IPs found at https://api.github.com/meta"
  }

  rule {
    action   = "deny(403)"
    priority = "2147483647"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Deny access to all IPs"
  }
}

resource "google_compute_url_map" "atlantis_url_map" {
  name            = "atlantis"
  default_service = module.atlantis-lb-https.backend_services["default"].self_link

  host_rule {
    hosts        = [var.domain]
    path_matcher = "allpaths"
  }

  path_matcher {
    name            = "allpaths"
    default_service = module.atlantis-lb-https.backend_services["default"].self_link

    path_rule {
      paths = [
        "/events"
      ]
      service = module.atlantis-lb-https.backend_services["events"].self_link
    }
  }
}
```

Finally, we should restrict the Cloud Run service's [ingress](https://cloud.google.com/run/docs/securing/ingress) to "Internal and Cloud Load Balancing". Then, the default domain cannot be used to bypass the protections added above.

### State

Atlantis keeps some state about which pull requests are open and the associated plans and locks. It also clones the Terraform repos in order to run the `plan` and `apply` commands.

Cloud Run uses an in-memory file system for disk storage. Atlantis' disk and memory usage is usually quite low so 8GB, Cloud Run's maximum, should be sufficient.

We can set Cloud Run to have a minimum number of instances of 1 and for the most part, this would mean we keep our state across invocations. However, occasionally, Cloud Run will restart the container causing all the built up state to be lost.

To get around this, we can backup the state to GCS with some regularity. Before starting the Atlantis server and after acquiring the lock described in [Runtime](#runtime), we would pull the state from GCS.

Let's expand on the startup script from before:

```shell
source /gcslock.sh
lock $ATLANTIS_BUCKET

function cleanup() {
  gsutil rsync -d -r $ATLANTIS_DATA_DIR gs://$ATLANTIS_BUCKET/atlantis/ || true
  unlock $ATLANTIS_BUCKET
}

trap 'cleanup' SIGTERM

# Pull from $ATLANTIS_BUCKET/atlantis to $ATLANTIS_DATA_DIR
mkdir -p $ATLANTIS_DATA_DIR
gsutil rsync -d -r gs://$ATLANTIS_BUCKET/atlantis/ $ATLANTIS_DATA_DIR
chown -R atlantis:atlantis $ATLANTIS_DATA_DIR

# start server...
export ATLANTIS_PORT=$PORT
gosu atlantis atlantis server &

# Sync the files back to GCS every minute
crontab -l | { cat; echo "* * * * * gsutil rsync -d -r $ATLANTIS_DATA_DIR gs://$ATLANTIS_BUCKET/atlantis/"; } | crontab -
crond

# Exit immediately when one of the background processes terminate.
wait -n
```

The addition of the `gsutil rsync` command in `cleanup` pushes any final state changes to GCS before finishing.

## Summary

We've seen how we can setup Atlantis in Cloud Run, taking care to configure the service to use always allocated CPU and introduce a locking mechanism to ensure only one instance is ever doing any work. We reasoned about why we'd need a load balancer and Cloud Armor and/or IAP to protect the endpoints. Finally, we looked into how we can keep state across restarts by periodically backing the state up to GCS. The diagram below summarises this architecture:

![Atlantis on Cloud Run GCP architecture](../atlantis-on-cloudrun-gcp-architecture.png)

### Reflection

#### GCS fuse

I recently stumbled onto [Using Cloud Storage FUSE with Cloud Run tutorial](https://cloud.google.com/run/docs/tutorials/network-filesystems-fuse) and this is what initially sparked this investigation.

Unfortunately, I wasn't able to get Atlantis working with gcsfuse. I believe the reason why not is related to Atlantis' use of [bbolt](https://github.com/etcd-io/bbolt) and how that works with the file system.

Periodically syncing the files to GCS was the compromise. It's conceivable that some state is missing or even corrupted in some way in GCS with this approach (e.g. if a container is `SIGKILL`ed halfway through an `rysnc`). If anything goes wrong, some manual intervention may be required to wipe the `/atlantis` directory in GCS. This is not a deal breaker though: for developers using Atlantis, they may see their plans and locks disappear so would need to ask Atlantis to run those again. 

In my experience with other services, Cloud Run does not restart always allocated CPU instances very frequently, with many days between restarts, so not having persistent state is a valid position to hold provided your team understands the consequences.

#### Compute Engine

With the locking and periodic GCS syncing workarounds, I am not entirely convinced running Atlantis on Cloud Run is a better solution than running it on Compute Engine. If we ask for 1 VM in Compute Engine, we will only ever have up to 1 VM running so no locking is necessary. Furthermore, attaching a data disk is sufficient for storing state across restarts. Fortunately, not all our investigations here are lost since the access control section would still apply by swapping the serverless backends for instance group ones that point to the Atlantis VM.

Earlier, I mentioned the need for an "ugly startup script" when deploying the Atlantis container image and data disk. This is because the Atlantis account has to `chown` the mounted directory which I couldn't find a good way to do at the time. Now I've experimented with this Cloud Run approach, which includes some additions to the existing Atlantis Docker image, I can probably find a nicer way to do that `chown` and roll with the Compute Engine setup instead of Cloud Run.

## Thanks for reading!