# 1. 개발 환경 구축하기 

- EC2 Instance
  - Type: t3.2xlarge (8vCPU, 32GiB)
  - Storage: 30GiB

## 설치 
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

# Add to .bashrc if not already present
grep -qxF 'export PATH=$PATH:/usr/local/go/bin' ~/.bashrc || echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
grep -qxF 'export GOPATH=$HOME/go' ~/.bashrc || echo 'export GOPATH=$HOME/go' >> ~/.bashrc
grep -qxF 'export PATH=$PATH:$GOPATH/bin' ~/.bashrc || echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc

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
vi init.sh
chmod +x  init.sh 
./init.sh
```

## 모듈 다운로드

```
mkdir -p $HOME/development/hashicorp/; cd $HOME/development/hashicorp/
git clone https://github.com/hashicorp/terraform-provider-aws.git
cd terraform-provider-aws
make tools 
```

## 빌드 및 Terraform 설정 

```
make build
```

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
