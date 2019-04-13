# Introduction

This playbook covers all the necessary components to deploy and manage your applications in the cloud.

The content you see here is a result of years of experience of DevOps Engineers with lots of trial-and-error.

This is a live document, all techniques and tools described here are in constant evolution and your contribution is welcomed.

# Infrastructure-as-code

## Principles

Deploy infrastructure the same way software is deployed.

Reusability is desired, but simplicity is priority: It's ok to duplicate code to avoid complexity.

## Terraform

Terraform has demonstrated being the best tool to deploy infrastructure. It supports most of the clouds and services, allows for a modular structure so reusability of code is possible and it's simple to learn and write code.

The only drawback it has (at least when comparing against services offered by the cloud provided like CloudFormation) is that you have to worry about state management. This problem is mitigated by following the conventions described later in this document.

Structure for Terraform is described below.

### Stacks

A Terraform stack is a repository with terraform definitions that manage a set of resources for a specific client.

Use one GIT repository per stack. Repository must be **private** as it is likely to contain sensitive data.

Add a prefix `infra-` to the stack to easily identify among your GIT repositories; Or `infra-<client name>-` when working with multiple clients within the same GIT team/group.

Stacks can be deployed to multiple times to different environments using Terraform Workspaces (see below).

Stacks can call modules to increase reusability.

Example file structure for a stack:

```
infra-ecs-cluster.git
├── _backend.tf      # backend declaration
├── _data.tf         # data types to fetch info
├── _outputs.tf      # outputs of this stack (optional)
├── _provider.tf     # provider(s) declaration
├── _variables.tf    # variables used by this stack
├── module-ecs.tf    # calling module 'ecs'
└── iam.tf           # defining IAM roles fed to the module
```

Add an underscore to .tf files that are common and required on all stacks so separate from tf files that describe resources.

### Modules

Use modules for re-usable components.

Modules should be open-source, unless there's a reason not to.

A module should be opinionated to deploy a component with a certain architecture. 

A module should have documentation on how to use it, inputs and outputs and examples.

If you reuire a variation of the module causing multiple resources to be conditional and making code and dependencies more complex, a new module should be created in this case.

Examples of modules:

* terraform-aws-ecs-alb
* terraform-aws-ecs
* terraform-aws-iam-ci-roles
* terraform-aws-network
* terraform-aws-audit
* terraform-gcp-gke-simple

A module should have its own GIT repository (usually public). Repository name should follow the format: `terraform-<main provider>-<module name>`

File structure of a module:

```
terraform-aws-ecs-alb.git
├── _data.tf         # data types to fetch info
├── _outputs.tf      # outputs of this module
├── _variables.tf    # variables used by this stack
├── alb.tf           # defines the ALB used with the ECS cluster
├── ecs.tf           # defines the ECS cluster and ASG for nodes
├── iam.tf           # defines IAM roles and policies for ECS
├── sg-alb.tf        # defines security groups used by the ALB
└── sg-ecs-nodes.tf  # defines security groups used by the nodes
```

Module outputs should contain all outputs of all resources declared by the module.

#### Granularity of Modules

Avoid modules too granular - modules that define just one or two resources.

Creating module to define a resource won't bring any benefit, as the resource can be declared directly in the stack with the same number of (or fewer) lines of code.

#### Modules calling modules

Avoid.

If a module needs to call another module, probably the module being called is too granular.

Prefer calling the module directly in the stack and feeding its outputs to the next module input.

Example:

Instead of an ECS module calling an IAM module to create the necessary roles, prefer either:
* ECS module declaring IAM roles directly as resources; or
* The stack calls the IAM module, pass its outputs as parameters to ECS module

### Workspaces

Use Terraform Workspaces to allow a stack to be deployed multiple times.

Naming:

`<account>-<region>-<environment>`

If there is only one environment (per account), use `default` as environment name.

If the stack span across multiple accounts, use `all` as the account name.

For workspaces that are not region specific or span across multiple regions, use `global` as region.

Examples:

|                 | Account Name | Region Name      | Environment Name |
| --------------- | ------------ | ---------------- | ---------------- |
| specific        | `nonprod`    | `us-west-2`      | `qa`             | 
| only one env    | `management` | `ap-southeast-2` | `default`        |
| across regions  | `prod`       | `global`         | `green`          |
| across accounts | `all`        | `us-east-2`      | `default`        |
| across both     | `all`        | `global`         | `default`        |

Even if you plan to deploy the stack only one time (one environment), still a good idea to use workspaces. Reason is if in the future another environment is required making workspaces necessary, it's not possible to migrate your deployed stack to a workspace.

# Containers

## Principles

Containers are used to keep consistency between environments (including local) and reuse tools during deplyoments.

A container should always carry the bare minimum to perform its purpose.

Always start with an Alpine container, or choose one that is based on Alpine.

## Scope

A good container image for performing AWS operations contains only AWS CLI, its requirements and maybe a bash script that calls it.

Another example, to build and deploy a serverless application, you will need (at least):
* A container image with serverless CLI
* Another container image to build that serverless function

We will see how the process works together later on this document.

## Repository Structure

A container image should have its own repository.

Set the repository as public, unless there's a reason not to.

Use DockerHub to build and host the image. DockerHub connects to GitHub and automatically builds images from Dockerfiles in repositores.

The repository should have a `Dockerfile` in the root.

Naming scheme:
* Repository: `docker-<image_name>`
* DockerHub: `<image-name>`

## Local vs Deployed

> The process described here is alpha (untested), any improvement is welcomed.

When developing locally with a docker container, the image used should be the same as deployed.

Sometimes it's not possible when running locally it requires tools for debugging or auto-refreshing that are not required when deployed to an environment.

In this case, use multi-stage builds so the same Dockerfile have stages for building, running locally and deployed.

* https://dev.to/brpaz/using-docker-multi-stage-builds-during-development-35bc

# Makefiles and the 3musketeers pattern

## Principles

Makefile is used as a common descriptor for building, deploying, testing and other operations to be performed.

By looking at the Makefile, the developer should have a quick view of all possible operations that can be performed on that code base, in the form of make targets.

Examples of targets:
* build
* test
* deploy

A developer should be able to clone a repository, set the required variables in the `.env` file and successfully run `make <target>` without any special setup in their machine other than having make, docker and docker-compose installed.

The main benefit of this pattern is running operations on both developers machine and CI worker the same.

## 3musketeers

A Makefile and docker-compose.yml file should exist in the repository.

Makefile have targets that call `docker-composer run <service>`. The service being a tool or the application itself.

Example:

```make
.env:
	cp .env.template .env

assumeRole: .env
	docker-compose run --rm aws assume-role.sh >> .env
.PHONY: assumeRole

deploy: assumeRole
	docker-compose run --rm deploy ./deploy.sh
.PHONY: deploy
```

The main target above, `deploy`, will first call `assumeRole`, which in turn calls `.env`.

* `.env` target creates the file if it doesn't exist.
* `assumeRole` uses AWS CLI to assume a role and output the credentials to the `.env` file
* `deploy` runs the deploy script using the credentials

The docker-compose.yml file for the example above looks like:

```yaml
version: '3.2'

services:
  deploy:
    image: dnxsolutions/aws:1.4.0
    entrypoint: "/bin/bash -c"
    env_file:
      - .env
    volumes:
      - ./deploy:/work

  aws:
    image: dnxsolutions/aws:1.4.0
    entrypoint: "/bin/bash -c"
    env_file: 
      - .env
```

In this example, the deploy script uses AWS CLI so we are using the `dnxsolutions/aws` image, same as the `assumeRole` target, which also uses the AWS CLI.

But in another case where we would deploy to kubernetes, the deploy service would use an image built for that, with kubectl, helm, etc.

# Application Deployment

# Pipeline-as-code

# Configuration Management