# Palo Alto VM-Series ARM Template

Azure ARM template for deploying 2 or 3 Palo Alto Networks VM-Series firewalls across selected availability zones for outbound and east-west traffic inspection, with internal load balancing.

This template is designed for Azure regions that support multiple availability zones. For regions that only offer availability sets, a more appropriate template should be used.

## Architecture

This template deploys the following resources:

- **Virtual Network** with 3 subnets (management, public, private) — or uses an existing VNET
- **3 VM-Series firewalls** (third firewall is optional), each with:
  - Management NIC (mgmt-subnet) — optional public IP
  - Public NIC (public-subnet) — outbound only
  - Private NIC (private-subnet) — connected to internal LB backend pool
- **Internal Load Balancer** with HA ports on the private subnet
- **NAT Gateway** (StandardV2 SKU) on the public subnet for outbound internet access
- **3 Network Security Groups** assigned at the subnet level:
  - `*-mgmt` — allows inbound SSH (22) and HTTPS (443) from a configurable prefix
  - `*-public` — allows all outbound, denies all inbound
  - `*-private` — allows all inbound and outbound

Each firewall is deployed into a separate availability zone (1, 2, 3) by default.

## Defaults

| Parameter | Default |
|---|---|
| VM Size | Standard_F16s_v2 |
| PAN-OS Version | 11.1.612 |
| OS Disk Type | Premium_LRS |
| VNET Prefix | 10.47.26.0/24 |
| Management Subnet | 10.47.26.128/26 |
| Public Subnet | 10.47.26.0/26 |
| Private Subnet | 10.47.26.64/26 |
| Internal LB Address | 10.47.26.100 |
| MGMT Allow Prefix | 10.0.0.0/8 |
| License Type | BYOL |
| Accelerated Networking | Enabled (all NICs) |
| Deploy Third Firewall | Yes |

## Deployment

### Azure Portal

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/)

Upload `arm-avzone-template-common_model-userdata.json` as a custom template in the Azure portal.

### Azure CLI

```bash
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file arm-avzone-template-common_model-userdata.json \
  --parameters username=<admin-user> password=<admin-password>
```

### PowerShell

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName <resource-group-name> `
  -TemplateFile arm-avzone-template-common_model-userdata.json `
  -username <admin-user> `
  -password <admin-password>
```

## Parameters

| Parameter | Type | Description |
|---|---|---|
| VNETOption | string | Create new VNET or use existing |
| ExistingVNETResourceGroup | string | Resource group of existing VNET (blank = same as firewall RG) |
| VNETName | string | Virtual network name |
| VNETPrefix | string | VNET address prefix |
| subnetName-Management | string | Management subnet name |
| subnetName-Public | string | Public (untrust) subnet name |
| subnetName-Private | string | Private (trust) subnet name |
| subnetPrefix-Management | string | Management subnet prefix |
| subnetPrefix-Public | string | Public subnet prefix |
| subnetPrefix-Private | string | Private subnet prefix |
| NATGatewayName | string | NAT Gateway name |
| internalLBName | string | Internal load balancer name |
| internalLBAddress | string | Internal LB frontend IP |
| healthProbePort | string | LB health probe TCP port (22, 80, 443) |
| FW1-Name | string | First firewall name |
| FW1-Zone | string | First firewall availability zone |
| FW2-Name | string | Second firewall name |
| FW2-Zone | string | Second firewall availability zone |
| deployThirdFirewall | string | Deploy a third firewall (Yes/No) |
| FW3-Name | string | Third firewall name |
| FW3-Zone | string | Third firewall availability zone |
| licenseType | string | BYOL, bundle1, or bundle2 |
| PANOSVersion | string | PAN-OS version (11.1.612 or latest) |
| VMSize | string | VM size for firewalls |
| OSDiskType | string | OS disk storage type |
| applyPublicIPToManagement | string | Assign public IP to management NIC (Yes/No) |
| NSGNamePrefix | string | Prefix for NSG names |
| MGMTAllowPrefix | string | Source prefix allowed to access management |
| username | string | Firewall admin username |
| password | securestring | Firewall admin password |
| optional-BootstrapCustomData | string | Bootstrap custom data string |

## Notes

- All resources deploy to the resource group's region. Ensure the resource group is in the desired region before deploying.
- When using an existing VNET, NSGs and the NAT Gateway are still created and associated with the subnets. Existing subnet properties (route tables, service endpoints) may be overwritten.
- Private IPs are dynamically assigned by Azure on all firewall interfaces.
- Accelerated networking is enabled on all NICs.
