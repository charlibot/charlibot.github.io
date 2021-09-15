---
title: "Nginx on Gce"
date: 2021-09-15T11:47:27+02:00
draft: true
---

This post captures thought process as much as the technical side. Skip to the end section for the code.

- Have some static content in GCS
- Served with storage backend + global load balancer (diagram)
- Has some excellent features. Used in day job and makes total sense to use in 99% of cases
- For my personal project with next to no traffic, would get an invoice from google every month charging me ~€20
- Not a crazy amount but enough to make me wince. 
- A whole conversation around cost vs benefit (spend 4 hours + lose some hair or just leave it)
- Also aware probably much better solutions for same price
- Read about GCP free tier, includes a e2-micro in one of three regions 
- Aware of Nginx and thought could use it as reverse proxy to gcp storage APIs
- In theory, brings costs down to essentially zero

How to do it?
- little more difficult than anticipated
- relatively little experience in Nginx
- eventually found an example that worked for me
- tried a bunch of variations with docker. Ended up using cloud-init because one less thing to concern myself with
  - cloud init is nice. I think can use it in other cloud providers (read some stuff around digital ocean for example when googling it)
- added ssl with certbot
  - examples on internet were a little outdated and I was too optimistic. terraform apply and hope
  - wasn't sure if should allow certbot to edit the nginx conf directly. no in the end, only adds a couple of lines and i'd rather know what's going on so used certonly 
  - hit rate limits from lets encrypt which are reasonable. Test with --staging! Have to wait a week
  - terraform to do the things
  - slow to iterate, always close but never quite there. Spend more time
- iptables was an annoying gotcha with cloud-init
- in summary, start an instance first and get it working with commands. Then, code it up. 