# What is Terraform

Terraform is an **infrastructure as config** tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can **version, reuse, and share**. 


Terraform creates and manages resources on cloud platforms and other services through their **application programming interfaces (APIs)**.

HashiCorp and the **Terraform community** have already written thousands of providers to manage many different types of resources and services.
You can find all publicly available providers on the [Terraform Registry](https://registry.terraform.io/?ajs_aid=f77d49d9-8c3b-4e1e-8c21-1f406a29ad31&product_intent=terraform), including Amazon Web Services (AWS), Azure, Google Cloud Platform (GCP), Kubernetes, Helm, GitHub, Splunk, DataDog, and many more.


The core Terraform workflow consists of three stages:
* **Write**: You define resources, which may be across multiple cloud providers and services. For example, you might create a configuration to deploy an application on virtual machines in a Virtual Private Cloud (VPC) network with security groups and a load balancer.
* **Plan**: Terraform creates an execution plan describing the infrastructure it will create, update, or destroy based on the existing infrastructure and your configuration.
* **Apply**: On approval, Terraform performs the proposed operations in the correct order, respecting any resource dependencies. For example, if you update the properties of a VPC and change the number of virtual machines in that VPC, Terraform will recreate the VPC before scaling the virtual machines.


## IBM acquired HashiCorp on April 24, 2024
 
IBM announced on April 24, 2024 that it will pay $6.4 billion ($35 per share) for HashiCorp, has stated that the company will remain independent after the takeover, but based on what actually happened after its previous acquisitions in the IT world, not everybody is sold on that promise.

Earlier in August 2023 Hashicorp, changed their licencing model to **Business Source License (BSL)** in place of the standard Mozilla Public License v2.0.  
In response to this **OpenTofu** was forked to be developed as a community-driven alternative to Terraform.

Developers always knew that to make money, Hashicorp was investing in their own cloud offering the "Infrastructure Cloud", rather than the previously opensource terraform project


## Terraform vs Others

**Chef, Puppet, and Ansible** are all configuration management tools, which means they are designed to **install and manage software on existing servers**.   
**CloudFormation, Heat, Pulumi, and Terraform** are provisioning tools, which means they are designed to provision the servers themselves (as well as the rest of your infrastructure, like load balancers, databases, networking configuration, etc), leaving the job of configuring those servers to other tools. 

My personal experience is with Terraform and CDK
| CDK | Terraform |
|---|---|
| CDK is more friendly to developers who likely already know Typescript, Python, etc. Things like abstraction and encapsulation, code reuse are easier with CDK since you get all the benefits of these first class languages | Terraform HCL is not hard to read or learn |
| CDK/Cloudformation drift detection just does not work as expected | Terraform State Management detects drift and can correct the change back to what the code/configuration defines| 
| loops and conditions, functions etc are easier with CDK because you are after all using python/typescript | loops and conditions can be complex in HCL |
| CDK uses a AWS developed lib called jsii, this marshals every non-node CDK call to call nodejs and return the result. | Terraform interacts with cloud services (AWS, Azure, GCP, etc.) via providers, which are separate executables written in Go. These providers implement Terraform's Plugin Protocol and handle the actual API calls to the cloud via standard HTtp requests |



These providers implement Terraform's Plugin Protocol and handle the actual API calls to the cloud.

### What does Terraform Cloud provide
Terraform Cloud is a platform developed by Hashicorp that helps with managing your Terraform code. It is used to enhance collaboration between developers and DevOps engineers, simplify your workflow and improve security overall around the product. 

* Terraform Run against workspace
A run in Terraform Cloud manages the lifecycle of a Terraform operation that is happening against your Workspace. The typical process a run goes through is:
  * Queuing → A run will be queued until it can be picked up by an available TFC worker
  * Planning → Run a terraform plan against your workspace
  * Cost Estimation → Shows you a cost estimate for your resources
  * Policy Checking → If policies are enabled for your workspace, it will check to see if anything is violating what you have put in place
  * Applying → If the plan and policy checking are successfully done, the code will get applied
* Drift monitoring
* Terraform Cloud also stores your variables securely, encrypting them at rest. 
* Terraform Cloud enables teams to work together with role-based access controls and policy enforcement.


## Terraform deep-dive

```text
$ terraform
Usage: terraform [global options] <subcommand> [args]
The available commands for execution are listed below.
The primary workflow commands are given first, followed by
less common or more advanced commands.
Main commands:
  init          Prepare your working directory for other commands
  validate      Check whether the configuration is valid
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure
```

At its core, terraform has pretty basic commands as shown above.

To deploy terraform on AWS, we start with
```terraform
terraform {
  required_providers {
    aws = {
      version = "~> 5.50.0"
    }
  }

  required_version = "~> 1.8.3"
}

provider "aws" {
  region = "eu-central-1"

  default_tags {
    tags = {
      Environment = "dev"
      Service     = "service-name"
      Project     = "demo-project"
    }
  }
}

```

The terraform {} block contains Terraform settings, including the required providers Terraform will use to provision your infrastructure. For each provider, the source attribute defines an optional hostname, a namespace, and the provider type. Terraform installs providers from the Terraform Registry by default. In this example configuration, the aws provider's source is defined as hashicorp/aws, which is shorthand for registry.terraform.io/hashicorp/aws.

You can also set a version constraint for each provider defined in the required_providers block. The version attribute is optional, but we recommend using it to constrain the provider version so that Terraform does not install a version of the provider that does not work with your configuration. If you do not specify a provider version, Terraform will automatically download the most recent version during initialization.


Now you can create resources in the format
```terraform
resource "<PROVIDER>_<TYPE>" "<NAME>" {
 [CONFIG …]
}
```

eg.
```terraform
resource "aws_s3_bucket" "bucket1" {
  bucket = var.bucket_name
}
```

Now you can use this "bucket" resource in other resources also, eg

```terraform
data "aws_iam_policy_document" "s3_policy" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.bucket1.arn}/*"]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.oai.iam_arn]
    }
  }
}

resource "aws_s3_bucket_policy" "static_website_bucket_policy" {
  bucket = aws_s3_bucket.bucket1.id
  policy = data.aws_iam_policy_document.s3_policy.json
}
```

Also, you can use variables to use across your terraform files 
* **Input Variables** serve as parameters for a Terraform module, so users can customize behavior without editing the source.
* **Output Values** are like return values for a Terraform module.
* **Local Values** are a convenience feature for assigning a short name to an expression.

When you declare variables in the root module of your configuration, you can set their values using CLI options and environment variables. When you declare them in child modules, the calling module should pass values in the module block.

```terraform
variable "bucket_name" {
 description = "This is the name of the s3 bucket which hosts nodejs app/website"
 type        = string
 default     = "ravi-test-chime"
}

output "website_url" {
  description = "Website URL (HTTPS)"
  value       = aws_cloudfront_distribution.distribution.domain_name
}

locals {
  service_name = "forum"
  owner        = "Community Team"
}
```

### data block
Data sources allow Terraform to use information defined outside of Terraform, defined by another separate Terraform configuration, or modified by functions. eg

```terraform

# find the latest AWS AMI available for ubuntu
data "aws_ami" "ubuntu" {
    most_recent = true
 
    filter {
        name   = "name"
        values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
    }

    filter {
        name   = "architecture"
        values = ["x86_64"]
    }
}

# create an instance with this AMI
resource "aws_instance" "ec2_instance" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.small"
 
  root_block_device {
    volume_size = 8
    volume_type = "gp3"
  }
}

```

### Loops in Terraform
Terraform provides multiple looping mechanisms, including:
1. **count** (for simple indexed loops)
2. **for_each** (for iterating over maps/sets)
3. for **expressions** (for transformations and filtering)



### Lets recap
* **resource**: A resource block declares a resource of a specific type with a specific **local name**. Terraform uses the name when referring to the resource in the same module, but it has no meaning outside that module's scope.
* **variables**: 
  - **input variable**:  user-provided parameters for a Terraform module
  - **output variable**: return values for a Terraform module
  - **data variable**:   short name to an expression for reuse
* **data**: A data block requests that Terraform read from a given data source ("aws_ami") and export the result under the given local name ("example"). The name is used to refer to this resource from elsewhere in the same Terraform module, but has no significance outside of the scope of a module.




### Terraform Plan

```terraform
$ terraform init
Initializing the backend...
Initializing provider plugins...
    - Reusing previous version of hashicorp/aws from the lock file
    - Using hashicorp/aws v4.19.0 from the shared cache directory
Terraform has been successfully initialized!
```

The terraform binary contains the basic functionality for Terraform, but it does not come with the code for any of the providers (e.g., the AWS Provider, Azure provider, GCP provider, etc.), so when you’re first starting to use Terraform, you need to run terraform init to tell Terraform to scan the code, figure out which providers you’re using, and download the code for them. By default, the provider code will be downloaded into a .terraform folder, which is Terraform’s scratch directory (you may want to add it to .gitignore). Terraform will also record information about the provider code it downloaded into a .terraform.lock.hcl file.



Now that you have the provider code downloaded, run the terraform plan command:

```terraform
$ terraform plan

(...)

Terraform will perform the following actions:

  # aws_instance.example will be created
  + resource "aws_instance" "example" {
      + ami                          = "ami-0fb653ca2d3203ac1"
      + arn                          = (known after apply)
      + associate_public_ip_address  = (known after apply)
      + availability_zone            = (known after apply)
      + cpu_core_count               = (known after apply)
      + cpu_threads_per_core         = (known after apply)
      + get_password_data            = false
      + host_id                      = (known after apply)
      + id                           = (known after apply)
      + instance_state               = (known after apply)
      + instance_type                = "t2.micro"
      + ipv6_address_count           = (known after apply)
      + ipv6_addresses               = (known after apply)
      + key_name                     = (known after apply)
      (...)
  }

Plan: 1 to add, 0 to change, 0 to destroy.
```


The plan command lets you see what Terraform will do before actually making any changes. This is a great way to sanity-check your code before unleashing it onto the world.
The output of the plan command is similar to the output of the diff command that is part of Unix, Linux, and git: anything with a plus sign (+) will be created, anything with a minus sign (–) will be deleted, and anything with a tilde sign (~) will be modified in place. In the preceding output, you can see that Terraform is planning on creating a single EC2 Instance and nothing else, which is exactly what you want.

To actually create the Instance, run the terraform apply command:

### Terraform state?

You might have noticed that every time you ran terraform plan or terraform apply, Terraform was able to find the resources it created previously and update them accordingly. 

Every time you run Terraform, it records information about what infrastructure it created in a **Terraform state file**. By default, when you run Terraform in the folder /foo/bar, Terraform creates the file /foo/bar/**terraform.tfstate**. This file contains a custom JSON format that records a mapping from the Terraform resources in your configuration files to the representation of those resources in the real world. 

If you want to use Terraform as a team, you run into several problems:
* **Shared storage for state files**. To be able to use Terraform to update your infrastructure, each of your team members needs access to the same Terraform state files. That means you need to store those files in a shared location.
* **Locking state files**. As soon as data is shared, you run into a new problem: locking. Without locking, if two team members are running Terraform at the same time, you can run into race conditions as multiple Terraform processes make concurrent updates to the state files, leading to conflicts, data loss, and state file corruption.
* **Isolating state files**. When making changes to your infrastructure, it’s a best practice to isolate different environments. For example, when making a change in a testing or staging environment, you want to be sure that there is no way you can accidentally break production. But how can you isolate your changes if all of your infrastructure is defined in the same Terraform state file?

### Shared storage for state files

You could always store the json based .tfstate files in git, but thats is not a recommended approach, because then 
1. the second problem of locking state files is not solved
2. and also, the .tfstate file can contains secrets like db username and password

Instead of using version control, the best way to manage shared storage for state files is to use Terraform’s built-in support for [remote backends](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)

Let's see we can use AWS S3 to store the state file and dynamoDB to lock the state file

```terraform
terraform {
  backend "s3" {
    # Replace this with your bucket name!
    bucket         = "terraform-up-and-running-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-2"

    # Replace this with your DynamoDB table name!
    dynamodb_table = "terraform-up-and-running-locks"
    encrypt        = true
  }
}
```

We need to reinitialize terraform 

```terraform
$ terraform init

Initializing the backend...
Acquiring state lock. This may take a few moments...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" 
  backend to the newly configured "s3" backend. No existing state 
  was found in the newly configured "s3" backend. Do you want to 
  copy this state to the new "s3" backend? Enter "yes" to copy and 
  "no" to start with an empty state.

  Enter a value:
```

```terraform
$ terraform apply

(...)

Acquiring state lock. This may take a few moments...

aws_dynamodb_table.terraform_locks: Refreshing state...
aws_s3_bucket.terraform_state: Refreshing state...

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Releasing state lock. This may take a few moments...

Outputs:

dynamodb_table_name = "terraform-up-and-running-locks"
s3_bucket_arn = "arn:aws:s3:::terraform-up-and-running-state"
```

###### A gotacha you need to be aware of
The backend block in Terraform does not allow you to use any variables or references. The following code will not work:

```terraform
# This will NOT work. Variables aren't allowed in a backend configuration.
terraform {
  backend "s3" {
    bucket         = var.bucket
    region         = var.region
    dynamodb_table = var.dynamodb_table
    key            = var.key
    encrypt        = true
  }
}
```

However, you can create a new file **backend.hcl** to store the arguments, like so

```terraform
# backend.hcl
bucket         = "terraform-up-and-running-state"
region         = "us-east-2"
dynamodb_table = "terraform-up-and-running-locks"
encrypt        = true
```

Only the key parameter remains in the Terraform code, since you still need to set a different key value for each module:
```terraform
# Partial configuration. The other settings (e.g., bucket, region) 
# will be passed in from a file via -backend-config arguments to 
# 'terraform init'
terraform {
  backend "s3" {
    key = "example/terraform.tfstate"
  }
}
```

### Isolating state files

The problem with defining all your infrastructure in a single set of terraform files, is that all of your Terraform state is now stored in a single file, too, and a mistake anywhere could break everything.  
You would always have different environments like dev and production, and you want to deploy resources seperately to these, and hence maintain different state files

There are two ways you could isolate state files:
* **Isolation via workspaces**: Useful for quick, isolated tests on the same configuration
* **Isolation via file layout**: Useful for production use cases for which you need strong separation between environments


##### Isolation via workspaces
**Terraform workspaces allow you to store your Terraform state in multiple, separate, named workspaces**. Terraform starts with a single workspace called “default,” and if you never explicitly specify a workspace, the default workspace is the one you’ll use the entire time. To create a new workspace or switch between workspaces, you use the terraform workspace commands.

```terraform
$ terraform workspace show
default
```
```terraform
$ terraform workspace new example1
Created and switched to workspace "example1"!

You're now on a new, empty workspace. Workspaces isolate their state, so if you run "terraform plan" Terraform will not see any existing state for this configuration.
```

However,   
* The state files for all of your workspaces are stored in the same backend (e.g., the same S3 bucket). That means you use the same authentication and access controls for all the workspaces, which is one major reason workspaces are an unsuitable mechanism for isolating environments (e.g., isolating staging from production).
* Workspaces are not visible in the code or on the terminal unless you run terraform workspace commands. When browsing the code, a module that has been deployed in one workspace looks exactly the same as a module deployed in 10 workspaces. This makes maintenance more difficult, because you don’t have a good picture of your infrastructure.

### Isolation via file layout
To achieve full isolation between environments, you need to do the following:
* Put the Terraform configuration files for each environment into a separate folder. For example, all of the configurations for the staging environment can be in a folder called stage and all the configurations for the production environment can be in a folder called prod.
* Configure a different backend for each environment, using different authentication mechanisms and access controls: e.g., each environment could live in a separate AWS account with a separate S3 bucket as a backend.
* Use Terraform modules to maintain common resources like buckets, etc


```terraform
module "website_s3_bucket" {
  source = "./modules/aws-s3-static-website-bucket"

  bucket_name = "var.bucket_name"
  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```


### Storing secrets in terraform

Obviously storing secrets in .tf file and commiting to your version control is a bad practice, consider the alternatives

##### 1. Use terraform variables use environment variables to set them before doing a terraform apply. eg

```terraform
variable "username" {
  description = "The username for the DB master user"
  type        = string
}
variable "password" {
  description = "The password for the DB master user"
  type        = string
}

# Set secrets via environment variables
export TF_VAR_username=(the username)
export TF_VAR_password=(the password)
# When you run Terraform, it'll pick up the secrets automatically
terraform apply
```

**Terraform 0.14 has added the ability to mark variables as sensitive, which helps keep them out of your logs, so you should add sensitive = true to both variables above!**


##### 2. Use Encrypted files and use aws_kms_secrets data to read them eg.

1. create a file db-creds.yml
```text
username: admin
password: password
```
**Do not commit this file into version control**
2. Encrypt this file with a aws kms key
```text
aws kms encrypt \
  --key-id <YOUR KMS KEY> \
  --region <AWS REGION> \
  --plaintext fileb://db-creds.yml \
  --output text \
  --query CiphertextBlob \
  > db-creds.yml.encrypted
```
3. Commit db-creds.yml.encrypted into version control
4. Use data block with "aws_kms_secret" to decrypt this
```
data "aws_kms_secrets" "creds" {
  secret {
    name    = "db"
    payload = file("${path.module}/db-creds.yml.encrypted")
  }
}

locals {
  db_creds = yamldecode(data.aws_kms_secrets.creds.plaintext["db"])
}
```
5. And then use these credentials
```terraform
resource "aws_db_instance" "example" {
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t2.micro"
  name                 = "example"
  # Set the secrets from the encrypted file
  username = local.db_creds.username
  password = local.db_creds.password
}
```

##### 3. Use services like Hashicorp Vault or AWS Secrets Manager

```terraform
data "aws_secretsmanager_secret_version" "creds" {
  # Fill in the name you gave to your secret
  secret_id = "db-creds"
}
locals {
  db_creds = jsondecode(
    data.aws_secretsmanager_secret_version.creds.secret_string
  )
}

```

I would obviously recommend the above approach, since you keep the secrets out of your version control
