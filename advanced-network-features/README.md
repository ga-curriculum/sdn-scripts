<h1>
  <span class="headline">SDN Scripts</span>
  <span class="subhead">Advanced Network Features in GCP</span>
</h1>

**Learning objective:** By the end of this lesson, students will be able to configure additional GCP networking features including custom routes, VPC peering, and load balancers to build more complex network designs.

## VPC Peering: Connecting Networks Together

``` hcl
# Create a second VPC for demonstration
resource "google_compute_network" "vpc_2" {
  name                    = "second-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "vpc_2_subnet" {
  name          = "vpc-2-subnet"
  ip_cidr_range = "10.1.0.0/24"  # Note: different range from our first VPC
  region        = var.region
  network       = google_compute_network.vpc_2.id
}

# Set up VPC peering between our networks
resource "google_compute_network_peering" "peering1" {
  name         = "peering-1-to-2"
  network      = google_compute_network.vpc.id
  peer_network = google_compute_network.vpc_2.id
  
  # Export custom routes (if you've defined any)
  export_custom_routes = true
}

resource "google_compute_network_peering" "peering2" {
  name         = "peering-2-to-1"
  network      = google_compute_network.vpc_2.id
  peer_network = google_compute_network.vpc.id
  
  # Import custom routes from the other VPC
  import_custom_routes = true
}
```

**Pro Tip:** VPC peering is non-transitive, meaning if VPC A is peered with VPC B, and VPC B is peered with VPC C, VPC A cannot communicate with VPC C through the peering connections.

## Creating Custom Routes

You can define custom routes to control how traffic flows within your network:

``` hcl
# Create a custom route
resource "google_compute_route" "custom_route" {
  name             = "custom-route"
  dest_range       = "192.168.0.0/24"  # Destination range
  network          = google_compute_network.vpc.id
  next_hop_gateway = "default-internet-gateway"
  priority         = 1000
}

# Route with next hop instance (for appliance-based routing)
resource "google_compute_route" "appliance_route" {
  name                   = "appliance-route"
  dest_range             = "10.2.0.0/16"
  network                = google_compute_network.vpc.id
  next_hop_instance      = google_compute_instance.public_vm.id
  next_hop_instance_zone = google_compute_instance.public_vm.zone
  priority               = 800
}
```

## HTTP Load Balancer Configuration

Let's set up a global HTTP load balancer to distribute traffic to your application:

``` hcl
# Create a health check
resource "google_compute_health_check" "default" {
  name               = "http-health-check"
  check_interval_sec = 5
  timeout_sec        = 5
  
  http_health_check {
    port = 80
  }
}

# Create an instance group (normally you'd use an instance group manager)
resource "google_compute_instance_group" "webservers" {
  name      = "webserver-group"
  zone      = "${var.region}-a"
  instances = [google_compute_instance.public_vm.id]
  
  named_port {
    name = "http"
    port = 80
  }
}

# Create backend service
resource "google_compute_backend_service" "default" {
  name          = "web-backend-service"
  port_name     = "http"
  protocol      = "HTTP"
  timeout_sec   = 10
  health_checks = [google_compute_health_check.default.id]
  
  backend {
    group = google_compute_instance_group.webservers.id
  }
}

# URL map to route requests to the backend
resource "google_compute_url_map" "default" {
  name            = "url-map"
  default_service = google_compute_backend_service.default.id
}

# HTTP proxy
resource "google_compute_target_http_proxy" "default" {
  name    = "http-proxy"
  url_map = google_compute_url_map.default.id
}

# Global forwarding rule (the frontend)
resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-rule"
  target     = google_compute_target_http_proxy.default.id
  port_range = "80"
}
```

## Shared VPC: For Multi-Project Setups

For larger organizations, Shared VPC lets you share a VPC network across multiple projects:

``` hcl
# You'd typically have a host project
resource "google_compute_shared_vpc_host_project" "host" {
  project = var.host_project_id
}

# And service projects that use the shared VPC
resource "google_compute_shared_vpc_service_project" "service1" {
  host_project    = var.host_project_id
  service_project = var.service_project_id
}
```

**Pro Tip:** Shared VPC is ideal for larger organizations where you want central IT to control networking, but allow individual teams to manage their own resources.

## Private Service Access

Connect privately to Google-managed services like Cloud SQL:

``` hcl
# Allocate an IP range for private service access
resource "google_compute_global_address" "private_ip_range" {
  name          = "private-service-access"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

# Create a private connection
resource "google_service_networking_connection" "private_service_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}

# Now you can create a private Cloud SQL instance
resource "google_sql_database_instance" "main" {
  name             = "private-database"
  database_version = "POSTGRES_13"
  region           = var.region
  
  depends_on = [google_service_networking_connection.private_service_connection]
  
  settings {
    tier = "db-f1-micro"
    
    ip_configuration {
      ipv4_enabled    = false  # No public IP
      private_network = google_compute_network.vpc.id
    }
  }
}
```

## Summary and Key Concepts

We've covered the essentials of Software-Defined Networking in Google Cloud:

1. **VPC Networks** are global and span all regions
2. **Subnets** are regional resources within your VPC
3. **Firewall Rules** control traffic flow between resources
4. **Cloud NAT** provides outbound internet access for private instances
5. **VPC Peering** connects separate VPC networks
6. **Load Balancers** distribute traffic to your applications
7. **Shared VPC** centralizes network management for multiple projects

The power of using Terraform is that you can:

- Define your entire network as code
- Version control your network configuration
- Deploy consistent environments with minimal effort
- Make changes predictably and safely
