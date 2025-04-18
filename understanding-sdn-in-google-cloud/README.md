<h1>
  <span class="headline">SDN Scripts</span>
  <span class="subhead">Understanding SDN in Google Cloud</span>
</h1>

**Learning Objective:** Understand the key components of Google Cloud networking and how they differ from traditional physical networks.

## What is software-defined networking?

Remember physical networking? The days of untangling cables, labeling switch ports, and trying to figure out which blinking light corresponds to which connection? Software-Defined Networking (SDN) changes all that by replacing physical components with virtual, programmable ones.

In Google Cloud Platform (GCP), networking is fully software-defined. There are no physical switches or routers for you to configure - instead, you define network resources through code or the Google Cloud Console, and GCP handles the physical infrastructure for you.

The real power comes when we use Terraform to manage these network resources. Instead of clicking through the console or running commands, we can define our entire network in code that's:

- Version-controlled (track changes over time)
- Repeatable (create identical environments)
- Testable (validate before deploying)
- Automated (no manual steps)

## Key GCP networking components

Here's how traditional networking concepts translate to GCP resources that we'll manage with Terraform:

| **Traditional Concept** | **GCP Resource** | **Terraform Resource**                  |
| ----------------------- | ---------------- | --------------------------------------- |
| Network                 | VPC Network      | `google_compute_network`                |
| Subnet                  | Subnet           | `google_compute_subnetwork`             |
| Routing Rules           | Routes           | `google_compute_route`                  |
| Firewall                | Firewall Rules   | `google_compute_firewall`               |
| Load Balancer           | Various LB Types | `google_compute_global_forwarding_rule` |
| NAT                     | Cloud NAT        | `google_compute_router_nat`             |

## Special characteristics of GCP networking

Unlike physical networks or even some other cloud providers, GCP has some unique characteristics:

1. **Global VPC Networks:** Your VPC network spans all GCP regions worldwide
2. **Regional Subnets:** Subnets are regional resources within your global VPC
3. **Automatic Mode vs. Custom Mode:** VPCs can automatically create subnets in each region (automatic) or let you define them manually (custom)
4. **Firewall Rules:** Deny by default, with explicit allow rules

## A Simple Network Architecture

Here's what we'll build in the following lessons:

**tktk - DAVID TO CREATE DIAGRAM**
