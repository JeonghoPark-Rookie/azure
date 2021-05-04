
main.tf

provider "azurerm" {
         features {}
}

resource1.tf
cat 
provider "azurerm" {
         features {}
}
root@user06admin:~/user06/vmss# cat resource1.tf
resource "azurerm_resource_group" "user06-rg01" {
    name     = "user06-rg01"
    location = "koreacentral"
    tags={environment = "Var Demo1"}
}

vmss1.tf

#3. myVnet01.tf

resource "azurerm_virtual_network" "user06-vnet1" {
    name                = "user06-myVnet1"
    address_space       = ["6.0.0.0/16"]
    location            = azurerm_resource_group.user06-rg01.location
    resource_group_name = azurerm_resource_group.user06-rg01.name
}

#4. mySubnet01.tf

resource "azurerm_subnet" "user06-subnet1" {
    name                 = "user06-mySubnet1"
    resource_group_name  = azurerm_resource_group.user06-rg01.name
    virtual_network_name = azurerm_virtual_network.user06-vnet1.name
    address_prefixes     = ["6.0.1.0/24"]
}

#5. public_ip01.tf

resource "azurerm_public_ip" "user06-PublicIp1" {
    name                = "myPublicip1"
    location            = azurerm_resource_group.user06-rg01.location
    resource_group_name = azurerm_resource_group.user06-rg01.name
    allocation_method   = "Static"
    domain_name_label   = azurerm_resource_group.user06-rg01.name                                                                                                                            
## 동일Region에추가Public IP 생성시에는아래내용수정(위설정했으면아래는#처리)
## domain_name_label   = "user06pubip2.westus.cloudapp.azure.com"
    tags = {
             environment = "staging" }
}

#6. myNetworkSecurityGroup01.tf

resource "azurerm_network_security_group" "user06nsg1" {
    name                      = "user06nsg1"
    location                   = azurerm_resource_group.user06-rg01.location
    resource_group_name = azurerm_resource_group.user06-rg01.name
    security_rule{
        name     = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
        }
    security_rule {
        name                       = "HTTP"
        priority                   = 2001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "80"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
        }

tags = {
    environment = "VAR Demo1"}
}

#7. lb01.tf

resource "azurerm_lb" "user06-lb1" {
    name                     = "user06lb1"
    location                 = azurerm_resource_group.user06-rg01.location
    resource_group_name      = azurerm_resource_group.user06-rg01.name
    frontend_ip_configuration {
        name                 = "user06PublicIPAddress"
        public_ip_address_id = azurerm_public_ip.user06-PublicIp1.id
    }
}

#8. lb_backendpool01.tf

resource "azurerm_lb_backend_address_pool" "user06-bpepool1" {
    resource_group_name = azurerm_resource_group.user06-rg01.name
    loadbalancer_id     = azurerm_lb.user06-lb1.id
    name                = "user06-BackEndAddressPool"
}

#9. lb_natpool01.tf

resource "azurerm_lb_nat_pool" "lbnatpool1" {
    resource_group_name            = azurerm_resource_group.user06-rg01.name
    name                           = "ssh"
    loadbalancer_id                = azurerm_lb.user06-lb1.id
    protocol                       = "Tcp"
    frontend_port_start            = 50000
    frontend_port_end              = 50119
    backend_port                   = 22
    frontend_ip_configuration_name = "user06PublicIPAddress"
}

#10. lb_probe01.tf

resource "azurerm_lb_probe" "user06-lb-probe1" {
    resource_group_name = azurerm_resource_group.user06-rg01.name
    loadbalancer_id     = azurerm_lb.user06-lb1.id
    name                = "http-probe"
    protocol            = "Http"
    request_path        = "/"
   port                 = 80
}

#11. lb_rule01.tf

resource "azurerm_lb_rule" "lbnatrule1" {
    resource_group_name            = azurerm_resource_group.user06-rg01.name
    loadbalancer_id                = azurerm_lb.user06-lb1.id
    name                           = "http"
    protocol                       = "Tcp"
    frontend_port                  = 80
    backend_port                   = 80
    backend_address_pool_id        = azurerm_lb_backend_address_pool.user06-bpepool1.id
    frontend_ip_configuration_name = "user06PublicIPAddress"
    probe_id                       = azurerm_lb_probe.user06-lb-probe1.id
}


#12. web.sh

# #!/bin/bash
# apt-get update -y
# apt-get install -y apache2
# echo "<html>" > /var/www/html/index.html   ## 리다이렉션반드시한개로
# echo "Hello World from $(hostname -f)" >> /var/www/html/index.html
# echo "</html>" >> /var/www/html/index.html

#13. vmss01.tf

resource "azurerm_virtual_machine_scale_set" "user06vmss1" {
    name                = "user06vmss1"
    location            = azurerm_resource_group.user06-rg01.location
    resource_group_name = azurerm_resource_group.user06-rg01.name
    upgrade_policy_mode = "Manual"

sku {
    name     = "Standard_D2_v3"
    tier     = "Standard"
    capacity = 2
}

storage_profile_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
}

storage_profile_os_disk {
    name              = ""
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
}

storage_profile_data_disk {
    lun           = 0
    caching       = "ReadWrite"
    create_option = "Empty"
    disk_size_gb  = 10
}

os_profile {
    computer_name_prefix = "vmss1"
    admin_username       = "asynch88"  ## VM 에접속할계정
    custom_data          = file("web1.sh")
}

os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
        path     = "/home/asynch88/.ssh/authorized_keys"   ## pwd실행후경로설정ex) /home/user06 등
        key_data = file("~/.ssh/id_rsa.pub")  ## VMSS.tf 생성전에터미널에서ssh-keygen 으로생성(엔터3번)
    }
}

network_profile {
    name     = "terraformnetworkprofile"
    primary  = true
    ip_configuration {
        name                                   = "TestIPConfiguration"
        primary                                = true
        subnet_id                              = azurerm_subnet.user06-subnet1.id
        load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.user06-bpepool1.id]
        load_balancer_inbound_nat_rules_ids    = [azurerm_lb_nat_pool.lbnatpool1.id]
        }
    network_security_group_id= azurerm_network_security_group.user06nsg1.id
    }

tags = {
    environment = "staging"
    }
}

resource2.rf

resource "azurerm_resource_group" "user06-rg02" {
    name     = "user06-rg02"
    location = "koreacentral"
    tags={environment = "Var Demo2"}
}


vmss2.tf

#3. myVnet02.tf

resource "azurerm_virtual_network" "user06-vnet2" {
    name                = "user06-myVnet2"
    address_space       = ["160.0.0.0/16"]
    location            = azurerm_resource_group.user06-rg02.location
    resource_group_name = azurerm_resource_group.user06-rg02.name
}

#4. mySubnet02.tf

resource "azurerm_subnet" "user06-subnet2" {
    name                 = "user06-mySubnet2"
    resource_group_name  = azurerm_resource_group.user06-rg02.name
    virtual_network_name = azurerm_virtual_network.user06-vnet2.name
    address_prefixes     = ["160.0.1.0/24"]
}

#5. public_ip02.tf

resource "azurerm_public_ip" "user06-PublicIp2" {
    name                = "myPublicip2"
    location            = azurerm_resource_group.user06-rg02.location
    resource_group_name = azurerm_resource_group.user06-rg02.name
    allocation_method   = "Static"
    domain_name_label   = azurerm_resource_group.user06-rg02.name                                                                                                                            
## 동일Region에추가Public IP 생성시에는아래내용수정(위설정했으면아래는#처리)
## domain_name_label   = "user06pubip2.westus.cloudapp.azure.com"
    tags = {
             environment = "staging" }
}

#6. myNetworkSecurityGroup02.tf

resource "azurerm_network_security_group" "user06nsg2" {
    name                      = "user06nsg2"
    location                   = azurerm_resource_group.user06-rg02.location
    resource_group_name = azurerm_resource_group.user06-rg02.name
    security_rule{
        name     = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
        }
    security_rule {
        name                       = "HTTP"
        priority                   = 2001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "80"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
        }

tags = {
    environment = "VAR Demo2"}
}

#7. lb02.tf

resource "azurerm_lb" "user06-lb2" {
    name                     = "user06lb2"
    location                 = azurerm_resource_group.user06-rg02.location
    resource_group_name      = azurerm_resource_group.user06-rg02.name
    frontend_ip_configuration {
        name                 = "user06PublicIPAddress"
        public_ip_address_id = azurerm_public_ip.user06-PublicIp2.id
    }
}

#8. lb_backendpool02.tf

resource "azurerm_lb_backend_address_pool" "user06-bpepool2" {
    resource_group_name = azurerm_resource_group.user06-rg02.name
    loadbalancer_id     = azurerm_lb.user06-lb2.id
    name                = "user06-BackEndAddressPool2"
}

#9. lb_natpool02.tf

resource "azurerm_lb_nat_pool" "lbnatpool2" {
    resource_group_name            = azurerm_resource_group.user06-rg02.name
    name                           = "ssh"
    loadbalancer_id                = azurerm_lb.user06-lb2.id
    protocol                       = "Tcp"
    frontend_port_start            = 50000
    frontend_port_end              = 50119
    backend_port                   = 22
    frontend_ip_configuration_name = "user06PublicIPAddress"
}

#10. lb_probe02.tf

resource "azurerm_lb_probe" "user06-lb-probe2" {
    resource_group_name = azurerm_resource_group.user06-rg02.name
    loadbalancer_id     = azurerm_lb.user06-lb2.id
    name                = "http-probe"
    protocol            = "Http"
    request_path        = "/"
   port                 = 80
}

#11. lb_rule02.tf

resource "azurerm_lb_rule" "lbnatrule2" {
    resource_group_name            = azurerm_resource_group.user06-rg02.name
    loadbalancer_id                = azurerm_lb.user06-lb2.id
    name                           = "http"
    protocol                       = "Tcp"
    frontend_port                  = 80
    backend_port                   = 80
    backend_address_pool_id        = azurerm_lb_backend_address_pool.user06-bpepool2.id
    frontend_ip_configuration_name = "user06PublicIPAddress"
    probe_id                       = azurerm_lb_probe.user06-lb-probe2.id
}


#12. web2.sh

# #!/bin/bash
# apt-get update -y
# apt-get install -y apache2
# echo "<html>" > /var/www/html/index.html   ## 리다이렉션반드시한개로
# echo "Hello World from $(hostname -f)" >> /var/www/html/index.html
# echo "</html>" >> /var/www/html/index.html

#13. vmss02.tf

resource "azurerm_virtual_machine_scale_set" "user06vmss2" {
    name                = "user06vmss2"
    location            = azurerm_resource_group.user06-rg02.location
    resource_group_name = azurerm_resource_group.user06-rg02.name
    upgrade_policy_mode = "Manual"

sku {
    name     = "Standard_D2_v3"
    tier     = "Standard"
    capacity = 2
}

storage_profile_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
}

storage_profile_os_disk {
    name              = ""
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
}

storage_profile_data_disk {
    lun           = 0
    caching       = "ReadWrite"
    create_option = "Empty"
    disk_size_gb  = 10
}

os_profile {
    computer_name_prefix = "vmss1"
    admin_username       = "asynch88"  ## VM 에접속할계정
    custom_data          = file("web2.sh")
}

os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
        path     = "/home/asynch88/.ssh/authorized_keys"   ## pwd실행후경로설정ex) /home/user06 등
        key_data = file("~/.ssh/id_rsa.pub")  ## VMSS.tf 생성전에터미널에서ssh-keygen 으로생성(엔터3번)
    }
}

network_profile {
    name     = "terraformnetworkprofile"
    primary  = true
    ip_configuration {
        name                                   = "TestIPConfiguration"
        primary                                = true
        subnet_id                              = azurerm_subnet.user06-subnet2.id
        load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.user06-bpepool2.id]
        load_balancer_inbound_nat_rules_ids    = [azurerm_lb_nat_pool.lbnatpool2.id]
        }
    network_security_group_id= azurerm_network_security_group.user06nsg2.id
    }

tags = {
    environment = "staging"
    }
}

web1.sh

#!/bin/bash
apt-get update -y
apt-get install -y apache2
echo "<html>" > /var/www/html/index.html   ## 리다이렉션반드시한개로
echo "<img src="https://user06cdnedp.azureedge.net/user06container/lake.jpg">" >> /var/www/html/index.html
echo "</html>" >> /var/www/html/index.html

web2.sh

#!/bin/bash
apt-get update -y
apt-get install -y apache2
echo "<html>" > /var/www/html/index.html   ## 리다이렉션반드시한개로
echo "<img src="https://user06cdnedp.azureedge.net/user06container/lake.jpg">" >> /var/www/html/index.html
echo "</html>" >> /var/www/html/index.html

peering.tf
## VNET peering을 위해 첫 번째 VNET에서 두 번째 VNET으로 peering 연결

resource "azurerm_virtual_network_peering"     "peer1" {
  name                      = "peer1to2"
  resource_group_name       = azurerm_resource_group.user06-rg01.name
  virtual_network_name      = azurerm_virtual_network.user06-vnet1.name
  remote_virtual_network_id = azurerm_virtual_network.user06-vnet2.id

}



## VNET peering을 위해 두 번째 VNET에서 첫 번째 VNET으로 peering 연결

resource "azurerm_virtual_network_peering"     "peer2" {
  name                      = "peer2to1"
  resource_group_name       = azurerm_resource_group.user06-rg02.name
  virtual_network_name      = azurerm_virtual_network.user06-vnet2.name
  remote_virtual_network_id = azurerm_virtual_network.user06-vnet1.id

}
