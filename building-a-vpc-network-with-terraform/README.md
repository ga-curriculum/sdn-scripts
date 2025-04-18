<h1>
  <span class="headline">SDN Scripts</span>
  <span class="subhead">Building a VPC Network with Terraform</span>
</h1>

**Learning objective:** By the end of this lesson, students will be able to create a complete VPC network with public and private subnets using Terraform for GCP.

## Setting up your Terraform configuration

Let's start by setting up the basic structure of our Terraform project:

```hcl
# provider.tf - Configure the Google provider
provider "google" {
  project = "your-project-id"
  region  = "us-central1"
  zone    = "us-central1-a"
}

# variables.tf - Define input variables
variable "project_id" {
  description = "The GCP project ID"
  type        = string
}

variable "region" {
  description = "The region to deploy resources"
  type        = string
  default     = "us-central1"
}

# We'll add more files for our network resources
```

## Creating the VPC Network

Now, let's create a custom-mode VPC network:

```hcl
# network.tf - Define VPC and subnets
resource "google_compute_network" "vpc" {
  name                    = "terraform-network"
  auto_create_subnetworks = false  # Custom mode - we'll create our own subnets
  
  # Note: In GCP, this creates a global VPC that spans all regions
}

# Create a public subnet
resource "google_compute_subnetwork" "public_subnet" {
  name          = "public-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.vpc.id
  
  # Enable flow logs for network monitoring
  log_config {
    aggregation_interval = "INTERVAL_5_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# Create a private subnet
resource "google_compute_subnetwork" "private_subnet" {
  name          = "private-subnet"
  ip_cidr_range = "10.0.2.0/24"
  region        = var.region
  network       = google_compute_network.vpc.id
  
  # Enable private Google access (allows VMs to reach Google APIs without 
  # a public IP)
  private_ip_google_access = true
  
  log_config {
    aggregation_interval = "INTERVAL_5_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}
```

> 🧠 By setting `auto_create_subnetworks = false`, we're using "custom mode" VPC, which gives us complete control over subnet creation rather than having GCP automatically create subnets in each region.

## Setting up cloud NAT for private instances

Instances in the private subnet won't have public IPs, so we need to set up Cloud NAT to allow them to access the internet:

```hcl
# nat.tf - Configure Cloud NAT for private instances
# First, create a Cloud Router
resource "google_compute_router" "router" {
  name    = "nat-router"
  region  = var.region
  network = google_compute_network.vpc.id
}

# Then, configure the NAT service
resource "google_compute_router_nat" "nat" {
  name                               = "nat-config"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "LIST_OF_SUBNETWORKS"
  
  subnetwork {
    name                    = google_compute_subnetwork.private_subnet.id
    source_ip_ranges_to_nat = ["ALL_IP_RANGES"]
  }
  
  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

## Configuring firewall rules

```hcl
# firewall.tf - Define firewall rules
# Allow SSH access to public instances
resource "google_compute_firewall" "allow_ssh_public" {
  name    = "allow-ssh-public"
  network = google_compute_network.vpc.name
  
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  
  source_ranges = ["0.0.0.0/0"]  # From anywhere (you may want to restrict this)
  target_tags   = ["public"]      # Apply to instances with this tag
}

# Allow internal communication between subnets
resource "google_compute_firewall" "allow_internal" {
  name    = "allow-internal"
  network = google_compute_network.vpc.name
  
  allow {
    protocol = "icmp"
  }
  
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  
  source_ranges = [
    google_compute_subnetwork.public_subnet.ip_cidr_range,
    google_compute_subnetwork.private_subnet.ip_cidr_range
  ]
}

# Allow web traffic to public instances
resource "google_compute_firewall" "allow_web" {
  name    = "allow-web"
  network = google_compute_network.vpc.name
  
  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }
  
  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web"]
}
```

**Security Note:** In GCP, the default firewall rule is to deny all incoming traffic. You need to explicitly create rules to allow traffic.

## Creating compute instances to test our network

```hcl
# instances.tf - Create sample VMs
# Public VM with external IP
resource "google_compute_instance" "public_vm" {
  name         = "public-vm"
  machine_type = "e2-medium"
  zone         = "${var.region}-a"
  
  tags = ["public", "web"]
  
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network    = google_compute_network.vpc.name
    subnetwork = google_compute_subnetwork.public_subnet.name
    
    access_config {
      // This gives the instance a public IP
    }
  }
}

# Private VM with no external IP
resource "google_compute_instance" "private_vm" {
  name         = "private-vm"
  machine_type = "e2-medium"
  zone         = "${var.region}-a"
  
  tags = ["private"]
  
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network    = google_compute_network.vpc.name
    subnetwork = google_compute_subnetwork.private_subnet.name
    // No access_config means no public IP
  }
}
```

## Outputs for important information

Finally, let's define some outputs to make it easier to reference important values:

```hcl
# outputs.tf - Define output values
output "vpc_name" {
  description = "The name of the VPC"
  value       = google_compute_network.vpc.name
}

output "public_subnet_name" {
  description = "Name of the public subnet"
  value       = google_compute_subnetwork.public_subnet.name
}

output "private_subnet_name" {
  description = "Name of the private subnet"
  value       = google_compute_subnetwork.private_subnet.name
}

output "public_vm_ip" {
  description = "Public IP of the web server"
  value       = google_compute_instance.public_vm.network_interface[0].access_config[0].nat_ip
}
```
