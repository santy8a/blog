---
title: "Inicio rápido: Creación de una máquina virtual Linux con infraestructura en Azure mediante terraform"
date: "2020-05-16T22:12:03.284Z"
description: "Aprenda a usar Terraform para crear y administrar un entorno completo de máquina virtual Linux en Azure."
---

Suscripción de Azure: si no tiene una suscripción a Azure, cree una cuenta gratuita de Azure antes de empezar.

Terraform permite definir y crear implementaciones completas de infraestructura de Azure. Se crean plantillas de Terraform en un formato legible para el usuario para crear y configurar recursos de Azure de forma coherente y reproducible. En este artículo se muestra cómo crear un entorno completo de Linux y recursos de apoyo con Terraform. También puede aprender a [instalar y configurar Terraform](install-configure.md).

## <a name="prerequisites"></a>Prerrequisitos

- Suscripción de Azure: si no tiene una suscripción a Azure, cree una cuenta gratuita de Azure antes de empezar.

## <a name="create-azure-connection-and-resource-group"></a>Creación de la conexión y el grupo de recursos de Azure

Analicemos las secciones de las plantillas de Terraform. También puede ver la versión completa de la [plantilla de Terraform](#complete-terraform-script) para copiarla y pegarla.

En la sección `provider` se indica a Terraform que utilice un proveedor de Azure. Para obtener los valores de `subscription_id`, `client_id`, `client_secret` y `tenant_id`, consulte [Instalación y configuración de Terraform](install-configure.md). 

> [!TIP]
> Si crea las variables de entorno para los valores o está usando la [experiencia de Bash de Azure Cloud Shell](/azure/cloud-shell/overview), no tendrá que incluir las declaraciones de variables en esta sección.

```hcl
provider "azurerm" {
    # The "feature" block is required for AzureRM provider 2.x. 
    # If you're using version 1.x, the "features" block is not allowed.
    version = "~>2.0"
    features {}
    
    subscription_id = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_secret   = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    tenant_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

En la sección siguiente, se crea un grupo de recursos denominado `myResourceGroup` en la ubicación `eastus`:

```hcl
resource "azurerm_resource_group" "myterraformgroup" {
    name     = "myResourceGroup"
    location = "eastus"

    tags = {
        environment = "Terraform Demo"
    }
}
```

En secciones adicionales, se hace referencia al grupo de recursos con `azurerm_resource_group.myterraformgroup.name`.

## <a name="create-virtual-network"></a>Creación de una red virtual

En la siguiente sección se crea una red virtual llamada `myVnet` en el espacio de direcciones `10.0.0.0/16`:

```hcl
resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "eastus"
    resource_group_name = azurerm_resource_group.myterraformgroup.name

    tags = {
        environment = "Terraform Demo"
    }
}
```

En la siguiente sección se crea una subred denominada `mySubnet` en la red virtual `myVnet`:

```hcl
resource "azurerm_subnet" "myterraformsubnet" {
    name                 = "mySubnet"
    resource_group_name  = azurerm_resource_group.myterraformgroup.name
    virtual_network_name = azurerm_virtual_network.myterraformnetwork.name
    address_prefix       = "10.0.2.0/24"
}
```


## <a name="create-public-ip-address"></a>Creación de una dirección IP pública

Para acceder a recursos de Internet, cree una dirección IP pública y asígnela a la VM. En la siguiente sección se crea una dirección IP pública llamada `myPublicIP`:

```hcl
resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = "myPublicIP"
    location                     = "eastus"
    resource_group_name          = azurerm_resource_group.myterraformgroup.name
    allocation_method            = "Dynamic"

    tags = {
        environment = "Terraform Demo"
    }
}
```

## <a name="create-network-security-group"></a>Creación de un grupo de seguridad de red

Los grupos de seguridad de red controlan el flujo del tráfico de red entrada y salida de la VM. En la siguiente sección se crea un grupo de seguridad de red llamado `myNetworkSecurityGroup` y se define una regla para permitir el tráfico de SSH en el puerto TCP 22:

```hcl
resource "azurerm_network_security_group" "myterraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = "eastus"
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
```


## <a name="create-virtual-network-interface-card"></a>Creación de una tarjeta de interfaz de red virtual

Las tarjetas de interfaz de red (NIC) virtuales conectan la VM a una red virtual determinada, una dirección IP pública y un grupo de seguridad de red. En la siguiente sección de una plantilla de Terraform se crea una NIC virtual llamada `myNIC` que se conecta a los recursos de red virtual que se hayan creado:

```hcl
resource "azurerm_network_interface" "myterraformnic" {
    name                        = "myNIC"
    location                    = "eastus"
    resource_group_name         = azurerm_resource_group.myterraformgroup.name

    ip_configuration {
        name                          = "myNicConfiguration"
        subnet_id                     = "azurerm_subnet.myterraformsubnet.id"
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = "azurerm_public_ip.myterraformpublicip.id"
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
```


## <a name="create-storage-account-for-diagnostics"></a>Creación de una cuenta de almacenamiento para el diagnóstico

Para almacenar el diagnóstico de arranque para una máquina virtual, se necesita una cuenta de almacenamiento. Estos diagnósticos de arranque pueden ayudarle a solucionar problemas y a supervisar el estado de la máquina virtual. La cuenta de almacenamiento que se crea solo sirve para almacenar datos de diagnóstico de arranque. Como las cuentas de almacenamiento deben tener un nombre único, con la siguiente sección se genera un texto aleatorio:

```hcl
resource "random_id" "randomId" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.myterraformgroup.name
    }
    
    byte_length = 8
}
```

Ahora puede crear una cuenta de almacenamiento. Con la siguiente sección se crea una cuenta de almacenamiento, con el nombre según el texto aleatorio generado en el paso anterior:

```hcl
resource "azurerm_storage_account" "mystorageaccount" {
    name                        = "diag${random_id.randomId.hex}"
    resource_group_name         = azurerm_resource_group.myterraformgroup.name
    location                    = "eastus"
    account_replication_type    = "LRS"
    account_tier                = "Standard"

    tags = {
        environment = "Terraform Demo"
    }
}
```


## <a name="create-virtual-machine"></a>Crear máquina virtual

El último paso es crear una máquina virtual y usar todos los recursos creados. En la siguiente sección se crea una máquina virtual llamada `myVM` y se asocia una NIC virtual llamada `myNIC`. Se usa la imagen `Ubuntu 16.04-LTS` más reciente y se crea un usuario llamado `azureuser` con la autenticación de contraseña deshabilitada.

 Los datos de la clave SSH se proporcionan en la sección `ssh_keys`. Proporcione una clave SSH pública en el campo `key_data`.

```hcl
resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = "myVM"
    location              = "eastus"
    resource_group_name   = azurerm_resource_group.myterraformgroup.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = "Standard_DS1_v2"

    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Premium_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "16.04.0-LTS"
        version   = "latest"
    }

    computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true
        
    admin_ssh_key {
        username       = "azureuser"
        public_key     = file("/home/azureuser/.ssh/authorized_keys")
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```

## <a name="complete-terraform-script"></a>Script completo de Terraform

Para reunir todas estas secciones y ver Terraform en acción, cree un archivo llamado `terraform_azure.tf` y pegue el siguiente contenido:

```hcl
# Configure the Microsoft Azure Provider
provider "azurerm" {
    # The "feature" block is required for AzureRM provider 2.x. 
    # If you're using version 1.x, the "features" block is not allowed.
    version = "~>2.0"
    features {}

    subscription_id = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_secret   = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    tenant_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}

# Create a resource group if it doesn't exist
resource "azurerm_resource_group" "myterraformgroup" {
    name     = "myResourceGroup"
    location = "eastus"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual network
resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "eastus"
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
    address_prefix       = "10.0.1.0/24"
}

# Create public IPs
resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = "myPublicIP"
    location                     = "eastus"
    resource_group_name          = azurerm_resource_group.myterraformgroup.name
    allocation_method            = "Dynamic"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "myterraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = "eastus"
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
    location                  = "eastus"
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
    location                    = "eastus"
    account_tier                = "Standard"
    account_replication_type    = "LRS"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = "myVM"
    location              = "eastus"
    resource_group_name   = azurerm_resource_group.myterraformgroup.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = "Standard_DS1_v2"

    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Premium_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "16.04.0-LTS"
        version   = "latest"
    }

    computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true
        
    admin_ssh_key {
        username       = "azureuser"
        public_key     = file("/home/azureuser/.ssh/authorized_keys")
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```


## <a name="build-and-deploy-the-infrastructure"></a>Compilación e implementación de la infraestructura

Una vez creada la plantilla de Terraform, el primer paso es inicializar Terraform. Este paso garantiza que Terraform cumple todos los requisitos previos para compilar la plantilla en Azure.

```bash
terraform init
```

El siguiente paso es que Terraform revise y valide la plantilla. En este paso se comparan los recursos solicitados con la información de estado que guarda Terraform y se genera la ejecución planeada. Los recursos de Azure no se crean en este momento.

```bash
terraform plan
```

Después de ejecutar el comando anterior, debería ver una pantalla similar a la siguiente:

```console
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


...

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.
  + azurerm_resource_group.myterraform
      <snip>
  + azurerm_virtual_network.myterraformnetwork
      <snip>
  + azurerm_network_interface.myterraformnic
      <snip>
  + azurerm_network_security_group.myterraformnsg
      <snip>
  + azurerm_public_ip.myterraformpublicip
      <snip>
  + azurerm_subnet.myterraformsubnet
      <snip>
  + azurerm_virtual_machine.myterraformvm
      <snip>
Plan: 7 to add, 0 to change, 0 to destroy.
```

Si todo parece correcto y está listo para compilar la infraestructura de Azure, aplique la plantilla en Terraform:

```bash
terraform apply
```

Cuando Terraform finalice, la infraestructura de máquina virtual estará lista. Obtenga la dirección IP pública de la máquina virtual con [az vm show](/cli/azure/vm):

```azurecli-interactive
az vm show --resource-group myResourceGroup --name myVM -d --query [publicIps] --o tsv
```

Después, puede acceder mediante SSH a la VM:

```bash
ssh azureuser@<publicIps>
```

## <a name="next-steps"></a>Pasos siguientes

> [!div class="nextstepaction"]
> [Más información sobre el uso de Terraform en Azure](/azure/terraform)