# Terraform
Infrastructure as Code (IaC) tools allow you to manage infrastructure with configuration files rather than through a graphical user interface. IaC allows you to build, change, and manage your infrastructure in a safe, consistent, and repeatable way by defining resource configurations that you can version, reuse, and share.

Terraform is HashiCorp's infrastructure as code tool. It lets you define resources and infrastructure in human-readable, declarative configuration files, and manages your infrastructure's lifecycle. Using Terraform has several advantages over manually managing your infrastructure:

Terraform can manage infrastructure on multiple cloud platforms.
The human-readable configuration language helps you write infrastructure code quickly.
Terraform's state allows you to track resource changes throughout your deployments.
You can commit your configurations to version control to safely collaborate on infrastructure.

###  Install Terraform
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```
### Verify the Installation
```
terraform -v
```
### Write configuration
```
touch main.tf
```

```hcl
terraform {
  required_providers {
    oci = {
      source = "oracle/oci"
    }
  }
}

provider "oci" {
  region              = "us-sanjose-1"
  auth                = "SecurityToken"
  config_file_profile = "learn-terraform"
}

resource "oci_core_vcn" "internal" {
  dns_label      = "internal"
  cidr_block     = "172.16.0.0/20"
  compartment_id = "<your_compartment_OCID_here>"
  display_name   = "My internal VCN"
}
```
### Terraform block
The `terraform {}` block contains Terraform settings, including the required providers Terraform will use to provision your infrastructure. For each provider, the `source` attribute defines an optional hostname, a namespace, and the provider type. Terraform installs providers from the Terraform Registry by default. In this example configuration, the `oci` provider's source is defined as oracle/oci, which is shorthand for `registry.terraform.io/oracle/oci`.

### Providers
The `provider` block configures the specified provider, in this case oci. A provider is a plugin that Terraform uses to create and manage your resources.

The `config_file_profile` attribute in the OCI provider block refers Terraform to the token credentials stored in the file that the OCI CLI created when you configured it. Never hard-code credentials or other secrets in your configuration files. Like other types of code, you may share and manage your Terraform configuration files using source control, so hard-coding secret values can expose them to attackers.

You can use multiple provider blocks in your Terraform configuration to manage resources from different providers. You can even use different providers together. For example, you could pass the IP address of an OCI instance to a monitoring resource from DataDog.
### Resources
Use `resource` blocks to define components of your infrastructure. A resource might be a physical or virtual component such as a VCN, or it can be a logical resource such as a Heroku application.

Resource blocks have two strings before the block: the resource type and the resource name. In this example, the resource type is `oci_core_vcn` and the name is `internal`. The prefix of the type maps to the name of the provider, in this case, oci. Together, the resource type and resource name form Terraform's unique ID for the resource. For example, the ID for your VCN is `oci_core_vcn.internal`.

Resource blocks contain arguments which you use to configure the resource. The example configuration contains arguments that set the DNS label, the CIDR block for the VCN, your OCID, and the display name.

### Initialize the directory
Terraform downloads the oci provider and installs it in a hidden subdirectory of your current working directory, named .terraform. The `terraform init` command prints out which version of the provider was installed. Terraform also creates a lock file named .terraform.lock.hcl which specifies the exact provider versions used, so that you can control when you want to update the providers used for your project.
```
terraform init
```

### Format and validate the configuration
The `terraform fmt` command automatically updates configurations in the current directory for readability and consistency.
```
terraform fmt
```
You can also make sure your configuration is syntactically valid and internally consistent by using the `terraform validate` command.
```
terraform validate
```

### Create infrastructure
Apply the configuration now with the `terraform apply` command. Before it applies any changes, Terraform prints out the execution plan which describes the actions Terraform will take in order to change your infrastructure to match the configuration.
```
terraform apply
```

### Inspect state
When you applied your configuration, Terraform wrote data into a file called `terraform.tfstate`. Terraform stores the IDs and properties of the resources it manages in this file, so that it can update or destroy those resources going forward.
```
terraform show
```
### Manually managing state
Terraform has a built-in command called terraform state for advanced state management. For example, you may want a list of the resources in your project's state, which you can get by using the list subcommand.
```
terraform state list
```

### Add a resource
Add a subnet to VCN in main.tf
```hcl
resource "oci_core_subnet" "dev" {
  vcn_id                      = oci_core_vcn.internal.id
  cidr_block                  = "172.16.0.0/24"
  compartment_id              = "<your_compartment_OCID_here>"
  display_name                = "Dev subnet 1"
  prohibit_public_ip_on_vnic  = true
  dns_label                   = "dev"
}
```

### Modify a resource
Terraform can modify existing resources as well as creating new ones. Modify the new subnet by changing it's display name from Dev subnet 1 to Dev subnet. Then save the file.
```hcl
  resource "oci_core_subnet" "dev" {
    vcn_id                      = oci_core_vcn.internal.id
    cidr_block                  = "172.16.0.0/24"
    compartment_id              = "<your_compartment_OCID_here>"
-   display_name                = "Dev subnet 1"
+   display_name                = "Dev subnet"
    prohibit_public_ip_on_vnic  = true
    dns_label                   = "dev"
  }
```
Notice the `~` prefix next to the subnet and its display name, indicating that Terraform can change this attribute of the resource. It also prints how many resources it will change, create, or destroy in the output.

Terraform can update some resources and attributes in-place, but others it can't change without destroying the resource. The prefix `-/+` would mean that Terraform planned to destroy and recreate the resource, rather than updating it in-place. Terraform handles these details for you, and the execution plan displays what Terraform will do.

### Destroy infrastructure
The `terraform destroy` command terminates resources defined in your Terraform configuration. This command is the reverse of terraform apply in that it terminates all the resources specified by the configuration. It does not destroy resources running elsewhere that are not described in the current configuration.
```
terraform destroy
```

### Declare variables
The configuration includes a number of hard-coded values. Terraform variables allow you to write configuration that is flexible and easier to re-use.
Declare variable to define your compartment ID and region so that you don't have to repeat these values in the configuration.
Create a new file called `variables.tf` containing the following configuration blocks.

```hcl
variable "compartment_id" {
  description = "OCID from your tenancy page"
  type        = string
}
variable "region" {
  description = "region where you have OCI tenancy"
  type        = string
  default     = "us-sanjose-1"
}
```

### Define the variables
Create a file called `terraform.tfvars` containing the following contents. This file will define values for the variables you just declared. As a best practice, Terraform configuration directories usually contain a version control configuration file that excludes files with `.tfvars` extensions from version control. Do not include sensitive values in Terraform configuration files that you could accidentally commit to version control.

```hcl
compartment_id   = "<your_compartment_OCID_here>"
region           = "us-sanjose-1"
```

### Reference the variables

```hcl
  terraform {
    required_providers {
      oci = {
        source = "oracle/oci"
      }
    }
  }

  provider "oci" {
-   region              = "us-sanjose-1"
+   region              = var.region
    auth                = "SecurityToken"
    config_file_profile = "learn-terraform"
  }

  resource "oci_core_vcn" "internal" {
    dns_label      = "internal"
    cidr_block     = "172.16.0.0/20"
-   compartment_id = "ocid1.tenancy.oc1...."
+   compartment_id = var.compartment_id
    display_name   = "My internal VCN"
  }

  resource "oci_core_subnet" "dev" {
    vcn_id                     = oci_core_vcn.internal.id
    cidr_block                 = "172.16.0.0/24"
-   compartment_id              = "ocid1.tenancy.oc1...."
+   compartment_id              = var.compartment_id
    display_name               = "Dev subnet"
    prohibit_public_ip_on_vnic = true
    dns_label                  = "dev"
  }
```

### Query data with outputs
Create a file called `outputs.tf`

```hcl
output "vcn_state" {
  description = "The state of the VCN."
  value       = oci_core_vcn.internal.state
}

output "vcn_cidr" {
  description = "CIDR block of the core VCN"
  value       = oci_core_vcn.internal.cidr_block
}
```

### Inspect output values
Terraform prints output values to the screen when you apply your configuration. Query the outputs with the `terraform output` command.

```
terraform output
```
