---
title: Using Visual Studio Code Remote Development in GCE over IAP
excerpt: What is ThreadLocal and why is it important? It's one of the key pieces of glue logic in modern Java web applications, so read on.
date: '2023-02-08T07:00:00.000-06:00'
tags:
- gcp
commments: true
---

Visual Studop Code has a very useful functionality to do Remote Development over SSH.
It is sometimes useful to run code directly on a Google Compute Engine (GCE) instance,
as it allows connectivity from inside a VPC (Virtual Private Cloud) subnetwork.
However, due to security concerns, many times our test instances will not
have a public IP that we can use to SSH to.
GCP provides a mechanism to connect to instances that lack a public IP via IAP (Identity Aware Proxy).
With GCE and IAP we can enjoy Remote Development even if our GCE instance lacks a public IP.


First, ensure that IAP is enabled for GCE as per the [docs](https://cloud.google.com/iap/docs/enabling-compute-howto).

Connect to your instance using the `gcloud` command:

    gcloud compute ssh --zone "INSTANCE_ZONE" "INSTANCE_NAME"  --tunnel-through-iap --project "PROJECT_NAME"

Replace the placeholders with your information:

* `INSTANCE_NAME`: Name of your instance.
* `INSTANCE_ZONE`: Zone where your instance resides.
* `PROJECT_NAME`: Project where your instance resides.

Connecting once using the `gcloud` command will ensure that the proper SSH keys are created in your `.ssh` directory.
We're going to use those in the next step.

Add a snippet to your `~/.ssh/config` file, telling SSH to use IAP when connecting to your host:

```
Host INSTANCE_NAME
    IdentityFile ~/.ssh/google_compute_engine
    User USERNAME
    HostName INSTANCE_NAME
    ProxyCommand gcloud compute start-iap-tunnel %h 22 --listen-on-stdin --zone INSTANCE_ZONE
```

Replace the placeholders with your information:

* `INSTANCE_NAME`: Name of your instance.
* `INSTANCE_ZONE`: Zone where your instance resides.
* `USERNAME`: Your username.

Add this snippet, and test by using `ssh` directly:

    ssh INSTANCE_NAME

You should be able to connect to the instance.

You can the follow the [official documentation](https://code.visualstudio.com/docs/remote/ssh) on how to use Remote Development.
