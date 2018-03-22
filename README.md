{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountPrefix": {
      "type": "string",
      "metadata": {
        "description": "Unique namespace for the Storage Account where the Virtual Machine's disks will be placed"
      }
    },
    "virtualNetworkName": {
      "type": "string",
      "defaultValue": "myVNET",
      "metadata": {
        "description": "Name of the virtual network provisioned for the cluster"
      }
    },
    "adminUsername": {
      "type": "string",
      "metadata": {
        "description": "Administrator user name used when provisioning virtual machines"
      }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Administrator password used when provisioning virtual machines"
      }
    },
    "dnsName": {
      "type": "string",
      "metadata": {
        "description": "Domain name for the publicly accessible Jenkins master node"
      }
    },
    "masterVmSize": {
      "type": "string",
      "defaultValue": "Standard_D3",
      "allowedValues": [
        "Standard_D3",
        "Standard_D4"
      ],
      "metadata": {
        "description": "The size of the virtual machines used when provisioning the Jenkins master node"
      }
    },
    "slaveNodes": {
      "type": "int",
      "defaultValue": 1,
      "metadata": {
        "description": "Number of Jenkins slave node (1 is the default)"
      }
    },
    "slaveVmSize": {
      "type": "string",
      "defaultValue": "Standard_D3",
      "allowedValues": [
        "Standard_D3",
        "Standard_D4"
      ],
      "metadata": {
        "description": "The size of the virtual machines used when provisioning Jenkins slave node(s)"
      }
    }
  },
  "variables": {
    "templateBaseUrl": "https://comvault.blob.core.windows.net/shared/",
    "sharedTemplateUrl": "[concat(variables('templateBaseUrl'), 'Shared-resoureces.json')]",
    "jenkMasterTemplateUrl": "[concat(variables('templateBaseUrl'), 'master.json')]",
    "jenkSlaveTemplateUrl": "[concat(variables('templateBaseUrl'), 'Slave.json')]",
    "namespace": "jenk",
    "networkSettings": {
      "virtualNetworkName": "[parameters('virtualNetworkName')]",
      "addressPrefix": "10.0.0.0/16",
      "subnet": {
        "dse": {
          "name": "dse",
          "prefix": "10.0.0.0/24",
          "vnet": "[parameters('virtualNetworkName')]"
        }
      },
      "statics": {
        "clusterRange": {
          "base": "10.0.0.",
          "start": 5
        },
        "jenkMaster": "10.0.0.240"
      }
    },
    "masterOsSettings": {
      "imageReference": {
        "publisher": "RedHat",
        "offer": "RHEL",
        "sku": "7.3",
        "version": "latest"
      },
      "scripts": [
        "https://comvault.blob.core.windows.net/test1/docker.sh"
      ]
    },
    "slaveOsSettings": {
      "imageReference": {
        "publisher": "RedHat",
        "offer": "RHEL",
        "sku": "7.3",
        "version": "latest"
      },
      "scripts": [
        "https://comvault.blob.core.windows.net/test1/Andoc.sh"
      ]
    },
    "sharedStorageAccountName": "[parameters('storageAccountPrefix')]"
  },
  "resources": [
    {
      "name": "shared",
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2015-01-01",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[variables('sharedTemplateUrl')]",
          "contentVersion": "1.0.0.0"
        },
        "parameters": {
          "networkSettings": {
            "value": "[variables('networkSettings')]"
          },
          "storageAccountName": {
            "value": "[variables('sharedStorageAccountName')]"
          }
        }
      }
    },
    {
      "name": "jenkMasterNode",
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2015-01-01",
      "dependsOn": [
        "[concat('Microsoft.Resources/deployments/', 'shared')]"
      ],
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[variables('jenkMasterTemplateUrl')]",
          "contentVersion": "1.0.0.0"
        },
        "parameters": {
          "storageAccountName": {
            "value": "[variables('sharedStorageAccountName')]"
          },
          "adminUsername": {
            "value": "[parameters('adminUsername')]"
          },
          "adminPassword": {
            "value": "[parameters('adminPassword')]"
          },
          "namespace": {
            "value": "[variables('namespace')]"
          },
          "vmbasename": {
            "value": "Master"
          },
          "subnet": {
            "value": "[variables('networkSettings').subnet.dse]"
          },
          "dnsname": {
            "value": "[parameters('dnsName')]"
          },
          "staticIp": {
            "value": "[variables('networkSettings').statics.jenkMaster]"
          },
          "vmSize": {
            "value": "[parameters('masterVmSize')]"
          },
          "slaveNodes": {
            "value": "[parameters('slaveNodes')]"
          },
          "osSettings": {
            "value": "[variables('masterOsSettings')]"
          }
        }
      }
    },
    {
      "name": "[concat('jenkSlaveNode', copyindex())]",
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2015-01-01",
      "dependsOn": [
        "[concat('Microsoft.Resources/deployments/', 'shared')]",
        "[concat('Microsoft.Resources/deployments/', 'jenkMasterNode')]"
      ],
      "copy": {
        "name": "vmLoop",
        "count": "[parameters('slaveNodes')]"
      },
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[variables('jenkSlaveTemplateUrl')]",
          "contentVersion": "1.0.0.0"
        },
        "parameters": {
          "storageAccountName": {
            "value": "[variables('sharedStorageAccountName')]"
          },
          "adminUsername": {
            "value": "[parameters('adminUsername')]"
          },
          "adminPassword": {
            "value": "[parameters('adminPassword')]"
          },
          "namespace": {
            "value": "[variables('namespace')]"
          },
          "vmbasename": {
            "value": "[concat('Slave', copyindex())]"
          },
          "masterNode": {
            "value": "[variables('networkSettings').statics.jenkMaster]"
          },
          "subnet": {
            "value": "[variables('networkSettings').subnet.dse]"
          },
          "vmSize": {
            "value": "[parameters('slaveVmSize')]"
          },
          "osSettings": {
            "value": "[variables('slaveOsSettings')]"
          }
        }
      }
    }
  ],
  "outputs": {}
}
