# Build infrastructure with Terraform

[Terraform](https://developer.hashicorp.com/terraform/intro "Intro to Terraform") is a tool that lets you create, change, and manage infrastructure using code. You write configuration files that describe your resources, and Terraform builds the infrastructure. This approach, called [infrastructure as code (IaC)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/infrastructure-as-code), makes infrastructure predictable, repeatable, and shareable across teams and environments.

In this tutorial, you use Terraform to run an NGINX web server in a Docker container. You follow the same workflow engineers use in production environments.

By the end of this tutorial, you will be able to:

- Define infrastructure in a Terraform configuration file
- Initialize a Terraform working directory
- Create and manage resources with `terraform apply`
- Remove resources with `terraform destroy`

> [!NOTE]
> This tutorial uses Docker to run infrastructure locally. The same Terraform workflow applies when managing infrastructure on cloud platforms like AWS, Azure, and Google Cloud.

## Prerequisites

Before you begin, install the following tools:

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/install-cli) v1.0.0 or later
- [Docker Desktop](https://www.docker.com/products/docker-desktop) v4.0.0 or later

Confirm Docker Desktop is running before you continue.

## Set up your project

Create a new directory for your Terraform project and navigate into it.

```shell
$ mkdir terraform-demo
$ cd terraform-demo
```

Next, create a configuration file called `main.tf`. Configuration files describe your infrastructure and tell Terraform what to build.

```shell
$ touch main.tf
```

Open `main.tf` in your text editor and paste in the following configuration:

```hcl
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_image" "nginx" {
  name = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "training"
  ports {
    internal = 80
    external = 80
  }
}
```

This configuration defines four blocks:

| **BLOCK** | **PURPOSE** |
|---|---|
| [`terraform {}`](https://developer.hashicorp.com/terraform/language/block/terraform) | Declares the required [provider](https://developer.hashicorp.com/terraform/language/providers) (Docker) and version constraint |
| [`provider "docker"`](https://developer.hashicorp.com/terraform/language/block/provider) | Configures Terraform to manage Docker on your local machine |
| [`resource "docker_image" "nginx"`](https://developer.hashicorp.com/terraform/language/block/resource) | Downloads the `nginx:latest` image from Docker Hub |
| [`resource "docker_container" "nginx"`](https://developer.hashicorp.com/terraform/language/block/resource) | Creates a running container from the image and maps port 80 to your machine |

Review the following details:

- **Version constraint:** `version = "~> 3.0"` allows any 3.x release of the provider. Specifying a version ensures your configuration works the same way every time. Without it, Terraform installs the latest provider, which may introduce breaking changes.
- **Image cleanup:** `keep_locally = false` tells Terraform to remove the NGINX image from your machine when you run `terraform destroy`. Set this to `true` if you want to keep the image locally after destroying the container.

> [!NOTE]
> **Windows users:** The `host` value `unix:///var/run/docker.sock` is the default Docker socket path on Linux and macOS. On Windows with Docker Desktop, use `npipe:////./pipe/docker_engine` instead.

## Initialize the working directory

When you create a new Terraform configuration, you must initialize the working directory. Initialization downloads the required providers and prepares Terraform to build your infrastructure.

Run `terraform init` to initialize your working directory and download the Docker provider plugin specified in your configuration.

```shell
$ terraform init
```

Terraform displays the following output, including the provider version it downloads.

```text
Initializing the backend...
Initializing provider plugins...
- Finding kreuzwerker/docker versions matching "~> 3.0"...
- Installing kreuzwerker/docker v3.6.2...
- Installed kreuzwerker/docker v3.6.2 (self-signed, key ID BD080C4571C6104C)
##...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
```

If you see any errors, confirm that Docker Desktop is running and try again.

To learn more about the files Terraform creates during initialization, see the [dependency lock file documentation](https://developer.hashicorp.com/terraform/language/files/dependency-lock).

## Provision the infrastructure

Run `terraform apply` to create the resources defined in your configuration.

```shell
$ terraform apply
```

Terraform generates an execution plan and asks for your confirmation:

```text
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # docker_container.nginx will be created
  + resource "docker_container" "nginx" {
      + image                                       = (known after apply)
      + name                                        = "training"
      ...
    }

  # docker_image.nginx will be created
  + resource "docker_image" "nginx" {
      + name         = "nginx:latest"
      ...
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 
```

Review the plan carefully. Each line beginning with `+` represents a resource Terraform will create. Enter `yes` to confirm.

Terraform builds your infrastructure:

```text
docker_image.nginx: Creating...
docker_image.nginx: Creation complete after 4s
docker_container.nginx: Creating...
docker_container.nginx: Creation complete after 1s

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Open a browser and go to [http://localhost:80](http://localhost:80). The NGINX welcome page confirms the container is running.

> [!NOTE]
> Terraform stores information about your infrastructure in a file named `terraform.tfstate`. It uses this state file to track resources and determine what changes to make when you run commands like `terraform apply` or `terraform destroy`.

> [!WARNING]
> The [state](https://developer.hashicorp.com/terraform/language/state) file may contain sensitive data. Do not commit it to version control.

## Destroy the infrastructure

When you no longer need infrastructure, you can destroy it to save resources, avoid extra costs, and reduce security risks.

Run `terraform destroy` to remove all resources that Terraform created in this tutorial.

```shell
$ terraform destroy
```

Terraform displays the resources it will remove and asks for your confirmation:

```text
Terraform will perform the following actions:

  # docker_container.nginx will be destroyed
  - resource "docker_container" "nginx" {
      - name = "training"
      ...
    }

  # docker_image.nginx will be destroyed
  - resource "docker_image" "nginx" {
      - name = "nginx:latest"
      ...
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: 
```

Enter `yes` to confirm. Terraform removes the NGINX container and image from your machine.

```text
docker_container.nginx: Destroying...
docker_container.nginx: Destruction complete after 1s
docker_image.nginx: Destroying...
docker_image.nginx: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.
```

> [!WARNING]
> `terraform destroy` permanently removes your infrastructure. Always review the plan carefully before entering `yes`.

## Next steps

In this tutorial, you wrote your first Terraform configuration, initialized a working directory, and used `terraform apply` and `terraform destroy` to create and remove a Docker container. These are the core commands engineers use to deploy and manage infrastructure in production. Defining infrastructure as code helps your team work faster, reduce errors, and replicate environments consistently.

Continue to the [Use input variables](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/docker-variables) tutorial to make your configuration more flexible and reusable by replacing hard-coded values with variables.

To build on what you learned:

- Read the [Terraform language documentation](https://developer.hashicorp.com/terraform/language) to learn the syntax used to write configuration files
- Explore the [kreuzwerker Docker provider](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs) to learn more about the Docker provider you used in this tutorial
- Review the [Terraform state documentation](https://developer.hashicorp.com/terraform/language/state) to learn how Terraform uses `terraform.tfstate` to track and manage your infrastructure
- Discover [Terraform use cases](https://developer.hashicorp.com/terraform/intro/use-cases) across different industries and infrastructure types
