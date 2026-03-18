# Build infrastructure with Terraform

[Terraform](https://developer.hashicorp.com/terraform/intro "Intro to Terraform") is an [infrastructure as code (IaC)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/infrastructure-as-code) tool that lets you define, provision, and manage infrastructure using configuration files instead of manual processes. Instead of clicking through a web console or running commands by hand, you write files that describe what your infrastructure should look like and let Terraform build it for you. This makes your infrastructure repeatable, auditable, and easy to share across a team.

In this tutorial, you will:

- Write a Terraform configuration file that provisions an NGINX web server in a Docker container
- Initialize a Terraform working directory and install the required provider
- Provision infrastructure with `terraform apply`
- Destroy infrastructure with `terraform destroy`

> [!NOTE]
> This tutorial uses Docker to run infrastructure locally on your machine. The same Terraform workflow applies when managing infrastructure on cloud platforms like AWS, Azure, and Google Cloud.

## Prerequisites

Before you begin, install the following tools:

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/install-cli) v1.0.0 or later
- [Docker Desktop](https://www.docker.com/products/docker-desktop) v4.0.0 or later

After installing Docker Desktop, open the application and confirm it shows **"Docker Desktop is running"** in the menu bar before continuing. Terraform communicates with Docker through a background process called the Docker daemon, and all commands will fail if Docker is not running.

## Set up your project

Create a new directory for your Terraform configuration and navigate into it.

```shell
$ mkdir terraform-demo
$ cd terraform-demo
```

Keep configuration files in their own directory. Terraform generates several files during initialization and planning, and an isolated directory keeps your project organized.

Next, create a file called `main.tf`. Terraform reads all `.tf` files in the working directory when it runs.

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

| Block | Purpose |
|---|---|
| `terraform {}` | Declares the required providers and version constraints for this configuration |
| `provider "docker"` | Configures the Docker [provider](https://developer.hashicorp.com/terraform/language/providers) and connects it to your local Docker daemon |
| `resource "docker_image" "nginx"` | Pulls the `nginx:latest` image from Docker Hub |
| `resource "docker_container" "nginx"` | Creates a running container from that image and maps port 80 |

Note the following before continuing:

- **Version constraint:** `version = "~> 3.0"` allows any 3.x release of the provider. We recommend always specifying a version constraint to make your configuration reproducible. Without one, Terraform installs the latest available version, which may introduce breaking changes.
- **Resource reference:** `docker_image.nginx.image_id` tells Terraform to use the image ID produced by the `docker_image` resource. Terraform reads this dependency automatically and always pulls the image before creating the container.
- **Image cleanup:** `keep_locally = false` tells Terraform to remove the NGINX image from your machine when you run `terraform destroy`. Set this to `true` if you want to keep the image locally after destroying the container.

> [!NOTE]
> **Windows users:** The `host` value `unix:///var/run/docker.sock` is the default Docker socket path on Linux and macOS. On Windows with Docker Desktop, use `npipe:////./pipe/docker_engine` instead.

## Initialize the working directory

Run `terraform init` to prepare your working directory. Terraform downloads the Docker provider plugin declared in your configuration.

```shell
$ terraform init
```

Terraform displays output similar to the following. We have truncated some of the output to save space.

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

Terraform creates a `.terraform/` directory in your project folder and downloads the provider plugin into it. It also creates a [`.terraform.lock.hcl`](https://developer.hashicorp.com/terraform/language/files/dependency-lock) file that records the exact provider version installed. Commit this file to version control so your team uses the same provider version.

If you see any errors, confirm that Docker Desktop is running and try again.

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

Review the plan carefully. Each line beginning with `+` represents a resource Terraform will create. When you are ready, type `yes` and press **Enter**.

```text
docker_image.nginx: Creating...
docker_image.nginx: Creation complete after 4s
docker_container.nginx: Creating...
docker_container.nginx: Creation complete after 1s

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Open a browser and navigate to http://localhost:80. The NGINX welcome page confirms the container is running and reachable on port 80.

> [!NOTE]
> Terraform writes the [state](https://developer.hashicorp.com/terraform/language/state) of your infrastructure to a file called `terraform.tfstate` in your working directory. Terraform uses this file to track which resources it manages so it can update or destroy them later. Treat this file as sensitive, as it may contain passwords, keys, or other secrets depending on your provider. Do not commit it to version control.

## Destroy the infrastructure

Run `terraform destroy` to remove all resources that Terraform created.

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

Type `yes` and press **Enter**. Terraform removes the container and image.

```text
docker_container.nginx: Destroying...
docker_container.nginx: Destruction complete after 1s
docker_image.nginx: Destroying...
docker_image.nginx: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.
```

> [!Warning]
> `terraform destroy` permanently removes all resources Terraform manages in the current working directory. In a production environment, always review the destruction plan before confirming.

## Next steps

In this tutorial, you provisioned and destroyed a Docker container using Terraform. Managing infrastructure as code means your infrastructure is now repeatable, reviewable, and consistent, eliminating the manual steps and configuration drift that cause problems in production environments.

Continue to the [Use input variables](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/docker-variables) tutorial to make your configuration more flexible and reusable by replacing hard-coded values with variables.

To go deeper on the concepts introduced in this tutorial:

- Read the [Terraform language documentation](https://developer.hashicorp.com/terraform/language) to learn HCL syntax, expressions, and built-in functions
- Explore the [kreuzwerker Docker provider](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs) for the full reference of the Docker provider used in this tutorial
- Review the [Terraform state documentation](https://developer.hashicorp.com/terraform/language/state) to learn how Terraform uses `terraform.tfstate` to track and manage your infrastructure
- Discover [Terraform use cases](https://developer.hashicorp.com/terraform/intro/use-cases) across different industries and infrastructure types
