# 2주차
## Azure랑 친해지기

[vm server 생성](https://cwal.tistory.com/85)
[azure 환경설정](https://cwal.tistory.com/83)


이 자료를 사용해서 Azure 회원가입, 테라폼 환경설정까지 성공했습니다.

![image](https://github.com/user-attachments/assets/df06e008-63ed-4f6b-a87c-fa20001c255a)
![image](https://github.com/user-attachments/assets/d84c5855-72ed-4242-8bfe-da4213615bfb)


## 실습 코드
```terraform
// https://cwal.tistory.com/85

# Configure the Microsoft Azure Provider
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "~>2.0"
    }
  }
}
provider "azurerm" {
  features {}
}

# Create a resource group if it doesn't exist
resource "azurerm_resource_group" "myterraformgroup" {
    name     = "myResourceGroup"
    location = "koreacentral"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual network
resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "koreacentral"
    resource_group_name = azurerm_resource_group.myterraformgroup.name

    tags = {
        environment = "Terraform Demo"
    }
}

# Create subnet
resource "azurerm_subnet" "myterraformsubnet" {
    name                 = "mySubnet"
    resource_group_name  = azurerm_resource_group.myterraformgroup.name
    virtual_network_name = azurerm_virtual_network.myterraformnetwork.name
    address_prefixes       = ["10.0.0.0/24"]
}

# Create public IPs
resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = "myPublicIP"
    location                     = "koreacentral"
    resource_group_name          = azurerm_resource_group.myterraformgroup.name
    allocation_method            = "Dynamic"

    tags = {
        environment = "Terraform Demo"
    }
}

output "vm_public_ip" {
    value = "${azurerm_public_ip.myterraformpublicip.*.ip_address}"
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "myterraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = "koreacentral"
    resource_group_name = azurerm_resource_group.myterraformgroup.name

    security_rule {
        name                       = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    tags = {
        environment = "Terraform Demo"
    }
}

# Create network interface
resource "azurerm_network_interface" "myterraformnic" {
    name                      = "myNIC"
    location                  = "koreacentral"
    resource_group_name       = azurerm_resource_group.myterraformgroup.name

    ip_configuration {
        name                          = "myNicConfiguration"
        subnet_id                     = azurerm_subnet.myterraformsubnet.id
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = azurerm_public_ip.myterraformpublicip.id
    }

    tags = {
        environment = "Terraform Demo"
    }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
    network_interface_id      = azurerm_network_interface.myterraformnic.id
    network_security_group_id = azurerm_network_security_group.myterraformnsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "randomId" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.myterraformgroup.name
    }

    byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "mystorageaccount" {
    name                        = "diag${random_id.randomId.hex}"
    resource_group_name         = azurerm_resource_group.myterraformgroup.name
    location                    = "koreacentral"
    account_tier                = "Standard"
    account_replication_type    = "LRS"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create (and display) an SSH key
resource "tls_private_key" "example_ssh" {
  algorithm = "RSA"
  rsa_bits = 4096
}
output "tls_private_key" { 
    value = tls_private_key.example_ssh.private_key_pem 
    sensitive = true
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = "myVM"
    location              = "koreacentral"
    resource_group_name   = azurerm_resource_group.myterraformgroup.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = "Standard_D2s_v3"

    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Premium_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
    }

    computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true

    admin_ssh_key {
        username       = "azureuser"
        public_key     = tls_private_key.example_ssh.public_key_openssh
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }

    tags = {
        environment = "Terraform Demo"
    }
}


/*
☁  aks [feat/terraform-tutorial-azure] ⚡  az vm list-skus --location koreacentral --size Standard_D --all --output table
ResourceType     Locations     Name                    Zones    Restrictions
---------------  ------------  ----------------------  -------  ----------------------------------------------------------------------------------------------------------------------------------------------------------
virtualMachines  KoreaCentral  Standard_D11_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D11_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D128lds_v6     1,3      None
virtualMachines  KoreaCentral  Standard_D128s_v6       1,3      None
virtualMachines  KoreaCentral  Standard_D12_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D12_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D13_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D13_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D14_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D14_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D15_v2         1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D16ads_v5      1,2,3    None
virtualMachines  KoreaCentral  Standard_D16ads_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D16alds_v6     2,3      None
virtualMachines  KoreaCentral  Standard_D16als_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D16as_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D16as_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D16as_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D16a_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16ds_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D16ds_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D16d_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16d_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16lds_v5      1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D16lds_v6      1,3      None
virtualMachines  KoreaCentral  Standard_D16ls_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D16s_v3        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16s_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16s_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D16s_v6        1,3      None
virtualMachines  KoreaCentral  Standard_D16_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D16_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D16_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D192s_v6       1,3      None
virtualMachines  KoreaCentral  Standard_D1_v2          1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D2ads_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D2ads_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D2alds_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D2als_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D2as_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D2as_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D2as_v6        2,3      None
virtualMachines  KoreaCentral  Standard_D2a_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2ds_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D2ds_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D2d_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2d_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2lds_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D2lds_v6       1,3      None
virtualMachines  KoreaCentral  Standard_D2ls_v5        1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D2s_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2s_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2s_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D2s_v6         1,3      None
virtualMachines  KoreaCentral  Standard_D2_v2          1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D2_v2_Promo    1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D2_v3          1,2,3    None
virtualMachines  KoreaCentral  Standard_D2_v4          1,2,3    None
virtualMachines  KoreaCentral  Standard_D2_v5          1,2,3    None
virtualMachines  KoreaCentral  Standard_D32ads_v5      1,2,3    None
virtualMachines  KoreaCentral  Standard_D32ads_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D32alds_v6     2,3      None
virtualMachines  KoreaCentral  Standard_D32als_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D32as_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D32as_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D32as_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D32a_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32ds_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D32ds_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D32d_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32d_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32lds_v5      1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D32lds_v6      1,3      None
virtualMachines  KoreaCentral  Standard_D32ls_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D32s_v3        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32s_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32s_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D32s_v6        1,3      None
virtualMachines  KoreaCentral  Standard_D32_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D32_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D32_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D3_v2          1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D3_v2_Promo    1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D48ads_v5      1,2,3    None
virtualMachines  KoreaCentral  Standard_D48ads_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D48alds_v6     2,3      None
virtualMachines  KoreaCentral  Standard_D48als_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D48as_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D48as_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D48as_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D48a_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48ds_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D48ds_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D48d_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48d_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48lds_v5      1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D48lds_v6      1,3      None
virtualMachines  KoreaCentral  Standard_D48ls_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D48s_v3        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48s_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48s_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D48s_v6        1,3      None
virtualMachines  KoreaCentral  Standard_D48_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D48_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D48_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4ads_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D4ads_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D4alds_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D4als_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D4as_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D4as_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D4as_v6        2,3      None
virtualMachines  KoreaCentral  Standard_D4a_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4ds_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D4ds_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D4d_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4d_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4lds_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D4lds_v6       1,3      None
virtualMachines  KoreaCentral  Standard_D4ls_v5        1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D4s_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4s_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4s_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D4s_v6         1,3      None
virtualMachines  KoreaCentral  Standard_D4_v2          1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D4_v2_Promo    1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D4_v3          1,2,3    None
virtualMachines  KoreaCentral  Standard_D4_v4          1,2,3    None
virtualMachines  KoreaCentral  Standard_D4_v5          1,2,3    None
virtualMachines  KoreaCentral  Standard_D5_v2          1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D5_v2_Promo    1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_D64ads_v5      1,2,3    None
virtualMachines  KoreaCentral  Standard_D64ads_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D64alds_v6     2,3      None
virtualMachines  KoreaCentral  Standard_D64als_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D64as_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D64as_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D64as_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D64a_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64ds_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D64ds_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D64d_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64d_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64lds_v5      1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D64lds_v6      1,3      None
virtualMachines  KoreaCentral  Standard_D64ls_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D64s_v3        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64s_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64s_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D64s_v6        1,3      None
virtualMachines  KoreaCentral  Standard_D64_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D64_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D64_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8ads_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D8ads_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D8alds_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D8als_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D8as_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D8as_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D8as_v6        2,3      None
virtualMachines  KoreaCentral  Standard_D8a_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8ds_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D8ds_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D8d_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8d_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8lds_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D8lds_v6       1,3      None
virtualMachines  KoreaCentral  Standard_D8ls_v5        1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D8s_v3         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8s_v4         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8s_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_D8s_v6         1,3      None
virtualMachines  KoreaCentral  Standard_D8_v3          1,2,3    None
virtualMachines  KoreaCentral  Standard_D8_v4          1,2,3    None
virtualMachines  KoreaCentral  Standard_D8_v5          1,2,3    None
virtualMachines  KoreaCentral  Standard_D96ads_v5      1,2,3    None
virtualMachines  KoreaCentral  Standard_D96ads_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D96alds_v6     2,3      None
virtualMachines  KoreaCentral  Standard_D96als_v6      2,3      None
virtualMachines  KoreaCentral  Standard_D96as_v4       1,2,3    None
virtualMachines  KoreaCentral  Standard_D96as_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D96as_v6       2,3      None
virtualMachines  KoreaCentral  Standard_D96a_v4        1,2,3    None
virtualMachines  KoreaCentral  Standard_D96ds_v5       1,2,3    None
virtualMachines  KoreaCentral  Standard_D96d_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D96lds_v5      1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D96lds_v6      1,3      None
virtualMachines  KoreaCentral  Standard_D96ls_v5       1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_D96s_v5        1,2,3    None
virtualMachines  KoreaCentral  Standard_D96s_v6        1,3      None
virtualMachines  KoreaCentral  Standard_D96_v5         1,2,3    None
virtualMachines  KoreaCentral  Standard_DS11-1_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS11_v2        1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS11_v2_Promo  1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS12-1_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS12-2_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS12_v2        1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS12_v2_Promo  1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS13-2_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS13-4_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS13_v2        1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS13_v2_Promo  1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS14-4_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS14-8_v2      1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS14_v2        1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS14_v2_Promo  1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS15_v2        1,2,3    NotAvailableForSubscription, type: Location, locations: KoreaCentral
virtualMachines  KoreaCentral  Standard_DS1_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS2_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS2_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS3_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS3_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS4_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS4_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS5_v2         1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']
virtualMachines  KoreaCentral  Standard_DS5_v2_Promo   1,2,3    ['NotAvailableForSubscription, type: Location, locations: KoreaCentral', 'NotAvailableForSubscription, type: Zone, locations: KoreaCentral, zones: 1,2,3']


*/
```
