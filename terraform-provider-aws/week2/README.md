# Development Environment Setup

## Environment
- EC2 Instance
  - Type: t3.2xlarge (8vCPU, 32GiB)
  - Storage: 30GiB

## Requirements 
- Terraform 0.12.26+ (to run acceptance tests)
- Go 1.23+ (to build the provider plugin)
- Mac, Linux or WSL (to build the provider plugin)

### Shell script for installing requirements

```
#!/bin/bash

# Update system packages using yum
sudo yum update -y

# Install build essentials, make, and other required dependencies
sudo yum groupinstall -y "Development Tools"
sudo yum install -y \
    make \
    wget \
    unzip \
    git \
    gcc \
    gcc-c++ \
    kernel-devel

# Verify Make installation
echo "Verifying Make installation:"
make --version

# Verify Git installation
echo "Verifying Git installation:"
git --version

# Download and install Terraform
TERRAFORM_VERSION="1.11.3"
wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/
rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip

# Verify Terraform installation
echo "Verifying Terraform installation:"
terraform version

# Download and install Go
GO_VERSION="1.23.9"
wget https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
rm go${GO_VERSION}.linux-amd64.tar.gz

# Set Go environment variables in both .bashrc and current session
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin

# Create basic Go workspace directories
mkdir -p $HOME/go/{bin,src,pkg}

# Reload shell configuration
source ~/.bashrc

# Verify all installations
echo "Verifying installations:"
echo "Make version:"
make --version

echo "Git version:"
git --version

echo "Terraform version:"
terraform version

echo "Go version:"
/usr/local/go/bin/go version

echo "GCC version:"
gcc --version

echo "Installation complete!"
echo "If Go is still not found, please run: source ~/.bashrc"
echo "Or logout and login again"
```

```
$ vi init.sh
$ chmod +x  init.sh 
$ ./init.sh
```

## Create the directory and clone the repository

```
$ mkdir -p $HOME/development/hashicorp/; cd $HOME/development/hashicorp/
$ git clone https://github.com/hashicorp/terraform-provider-aws.git
$ cd terraform-provider-aws
```

## Install the tools

```
$ make tools 
```

- [GNUmakefile](https://github.com/hashicorp/terraform-provider-aws/blob/main/GNUmakefile)

```
tools: prereq-go ## Install tools
	@echo "make: Installing tools..."
	cd .ci/providerlint && $(GO_VER) install .
	cd .ci/tools && $(GO_VER) install github.com/YakDriver/tfproviderdocs
	cd .ci/tools && $(GO_VER) install github.com/client9/misspell/cmd/misspell
	cd .ci/tools && $(GO_VER) install github.com/golangci/golangci-lint/v2/cmd/golangci-lint
	cd .ci/tools && $(GO_VER) install github.com/hashicorp/copywrite
	cd .ci/tools && $(GO_VER) install github.com/hashicorp/go-changelog/cmd/changelog-build
	cd .ci/tools && $(GO_VER) install github.com/katbyte/terrafmt
	cd .ci/tools && $(GO_VER) install github.com/pavius/impi/cmd/impi
	cd .ci/tools && $(GO_VER) install github.com/rhysd/actionlint/cmd/actionlint
	cd .ci/tools && $(GO_VER) install github.com/terraform-linters/tflint
	cd .ci/tools && $(GO_VER) install mvdan.cc/gofumpt
```

- modify the GO_VER
```
# GO_VER                       ?= $(shell echo go`cat .go-version | xargs`)
GO_VER                       ?= go
```

## Build the Provider

```
$ make build
```

- [GNUmakefile](https://github.com/hashicorp/terraform-provider-aws/blob/main/GNUmakefile)

```
build: prereq-go fmt-check ## Build provider
	@echo "make: Building provider..."
	@$(GO_VER) install
```

## Test the local provider with Terraform

### Create the main.tf and ~/.terraformrc files
- main.tf
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

# Create a VPC
resource "aws_vpc" "example" {
  cidr_block = "10.0.0.0/16"
}
```

-  ~/.terraformrc
```
provider_installation {
  dev_overrides {
    "hashicorp/aws" = "/home/ec2-user/go/bin"
  }
  direct {}
}
```
### Check the the provider binary built

```sh
$ ls -la $GOPATH/bin/terraform-provider-aws

-rwxr-xr-x. 1 ec2-user ec2-user 1014610856 May 18 14:48 /home/ec2-user/go/bin/terraform-provider-aws
```

### Run terraform apply without terraform init

```
$ terraform apply

╷
│ Warning: Provider development overrides are in effect
│ 
│ The following provider development overrides are set in the CLI configuration:
│  - hashicorp/aws in /home/ec2-user/go/bin
│ 
│ The behavior may therefore not match any released version of the provider and applying changes may cause the
│ state to become incompatible with published releases.
╵

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with
the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.example will be created
  + resource "aws_vpc" "example" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags_all                             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_vpc.example: Creating...
aws_vpc.example: Creation complete after 1s [id=vpc-0a010d1daf07cc3da]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```
