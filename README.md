# gcp-project

# Automating the Deployment of Infrastructure Using Deployment Manager

1. Open cloud shell and create a folder named dminfra.

   mkdir dminfra
   cd dminfra

2. Create a new file named config.yaml and define your network rosources

   imports:

   - path: instance-template.jinja

   resources:

   # Create the auto-mode network

   - name: mynetwork
     type: compute.v1.network
     properties:
     autoCreateSubnetworks: true

   # Create the firewall rule

   - name: mynetwork-allow-http-ssh-rdp-icmp
     type: compute.v1.firewall
     properties:
     network: \$(ref.mynetwork.selfLink)
     sourceRanges: ["0.0.0.0/0"]
     allowed: - IPProtocol: TCP
     ports: [22, 80, 3389] - IPProtocol: ICMP

     # Create the mynet-us-vm instance

   - name: mynet-us-vm
     type: instance-template.jinja
     properties:
     zone: us-central1-a
     machineType: n1-standard-1
     network: \$(ref.mynetwork.selfLink)
     subnetwork: regions/us-central1/subnetworks/mynetwork

   # Create the mynet-eu-vm instance

   - name: mynet-eu-vm
     type: instance-template.jinja
     properties:
     zone: europe-west1-d
     machineType: n1-standard-1
     network: \$(ref.mynetwork.selfLink)
     subnetwork: regions/europe-west1/subnetworks/mynetwork

3. Create a new file named instance-template.jinja and define your instance template resources

   resources:

   - name: {{ env["name"] }}
     type: compute.v1.instance
     properties:
     machineType: zones/{{ properties["zone"] }}/machineTypes/{{ properties["machineType"] }}
     zone: {{ properties["zone"] }}
     networkInterfaces: - network: {{ properties["network"] }}
     subnetwork: {{ properties["subnetwork"] }}
     accessConfigs: - name: External NAT
     type: ONE_TO_ONE_NAT
     disks: - deviceName: {{ env["name"] }}
     type: PERSISTENT
     boot: true
     autoDelete: true
     initializeParams:
     sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/family/debian-9

4. Create a deployment script named deployment.sh and add the deployment command

   gcloud deployment-manager deployments create dminfra --config=config.yaml --preview
   gcloud deployment-manager deployments update dminfra

5. run the deployment script

   . deployment.sh

Automating the Deployment of Infrastructure Using Terraform

1. Open cloud shell and confirm Terraform is installed

   terraform --version

2. Create a folder named tfinfra

   mkdir tfinfra
   cd tfinfra

3. Create a new file named mynetwork.tf and define your network rosources

   resource "google_compute_network" "mynetwork" {
   name = "mynetwork"
   auto_create_subnetworks = true
   }
   resource "google_compute_firewall" "mynetwork-allow-http-ssh-rdp-icmp" {
   name = "mynetwork-allow-http-ssh-rdp-icmp"
   network = google_compute_network.mynetwork.self_link
   allow {
   protocol = "tcp"
   ports = ["22", "80", "3389"]
   }
   allow {
   protocol = "icmp"
   }
   }
   module "mynet-us-vm" {
   source = "./instance"
   instance_name = "mynet-us-vm"
   instance_zone = "us-central1-a"
   instance_subnetwork = google_compute_network.mynetwork.self_link
   }
   module "mynet-eu-vm" {
   source = "./instance"
   instance_name = "mynet-eu-vm"
   instance_zone = "europe-west1-d"
   instance_subnetwork = google_compute_network.mynetwork.self_link
   }

4. Create a new file named provider.tf and define your cloud provider

provider "google" {}

5. Create a folder named instance

mkdir instance
cd instance

6.  Create a new file named main.tf and define your instance template

        variable "instance_name" {}
        variable "instance_zone" {}
        variable "instance_type" {
        default = "n1-standard-1"
        }
        variable "instance_subnetwork" {}

        resource "google_compute_instance" "vm_instance" {
        name         = "${var.instance_name}"
        zone         = "${var.instance_zone}"
        machine_type = "${var.instance_type}"
        boot_disk {
            initialize_params {
            image = "debian-cloud/debian-9"
            }
        }

    network_interface {
    subnetwork = "\${var.instance_subnetwork}"
    access_config { # Allocate a one-to-one NAT IP to the instance
    }
    }

}

7. Create a deployment script named deployment.sh and add the deployment command

   terraform fmt
   terraform init
   terraform plan
   terraform apply

8. run the deployment script

   . deployment.sh
