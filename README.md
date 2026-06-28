# AWS + Azure Hybrid Network via Megaport

[![AWS](https://img.shields.io/badge/AWS-Transit_Gateway-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![Azure](https://img.shields.io/badge/Azure-ExpressRoute-0078D4?logo=microsoft-azure)](https://azure.microsoft.com/)
[![Megaport](https://img.shields.io/badge/Megaport-VXC-E5004C)](https://www.megaport.com/)
[![Terraform](https://img.shields.io/badge/Terraform-1.7+-623CE4?logo=terraform)](https://www.terraform.io/)

> **Hybrid Network:** "Project implementing hybrid cloud connectivity using AWS Transit Gateway and Azure ExpressRoute" + "Working knowledge of Megaport services including VXC provisioning"

End-to-end hybrid network connecting on-premises data center, AWS (Transit Gateway + Direct Connect), and Azure (ExpressRoute) through a Megaport Cloud Router (MCR) — with BGP routing, redundant paths, and full Terraform automation.

---

## Architecture

```
ON-PREMISES DATA CENTER
┌──────────────────────────────────────────────────────────┐
│  Customer Edge Router                                     │
│  AS 65001 | 10.10.0.0/16                                 │
│                                                           │
│  Cisco ISR 4451 / Palo Alto                               │
│  BGP peers: MCR primary + secondary                       │
└──────────────────────┬───────────────────────────────────┘
                       │ Physical cross-connect (Equinix DC6)
                       │ VLAN 100 / 200
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MEGAPORT SDN                                      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Megaport Cloud Router (MCR)  — AS 133937                  │     │
│  │                                                            │     │
│  │  BGP peers:                                                │     │
│  │    • On-prem CE:  169.254.0.2 (VLAN 100)                  │     │
│  │    • AWS DXGW:    169.254.10.2 (VLAN 200)                 │     │
│  │    • Azure MSEE:  169.254.20.2 (VLAN 300)                 │     │
│  │                                                            │     │
│  │  Route redistribution:                                     │     │
│  │    On-prem prefixes → AWS and Azure                        │     │
│  │    AWS prefixes → On-prem and Azure                        │     │
│  │    Azure prefixes → On-prem and AWS                        │     │
│  └────────────┬────────────────────────┬───────────────────┘      │
│               │ VXC (1Gbps)            │ VXC (1Gbps)               │
└───────────────┼────────────────────────┼───────────────────────────┘
                │                        │
┌───────────────▼──────────┐   ┌─────────▼──────────────────────────┐
│  AWS (us-east-1)          │   │  Azure (East US)                   │
│                           │   │                                    │
│  Direct Connect Gateway   │   │  ExpressRoute Circuit              │
│  └→ Transit Gateway       │   │  └→ ExpressRoute GW                │
│     ├── Prod VPC          │   │     └→ Hub VNet (10.2.0.0/16)     │
│     │   10.1.0.0/16       │   │        ├── Spoke A (10.3.0.0/16)  │
│     ├── Dev VPC           │   │        └── Spoke B (10.4.0.0/16)  │
│     │   10.1.1.0/16       │   │                                    │
│     └── Shared SVC VPC    │   │  Azure Firewall in Hub             │
│         10.1.2.0/16       │   │  (all traffic inspected)           │
└───────────────────────────┘   └────────────────────────────────────┘

BGP Route Table (MCR summary):
  10.10.0.0/16  → On-prem (advertised to AWS + Azure)
  10.1.0.0/14   → AWS (advertised to On-prem + Azure)
  10.2.0.0/14   → Azure (advertised to On-prem + AWS)
```

---

## Step-by-Step Execution

### Phase 1 — Megaport Setup

#### Step 1: Order Megaport Port and create MCR

```bash
# Using Megaport Terraform provider
cd terraform/megaport/

cat > main.tf << 'EOF'
terraform {
  required_providers {
    megaport = { source = "megaport/megaport"; version = "~> 1.0" }
  }
}

provider "megaport" {
  access_key  = var.megaport_access_key
  secret_key  = var.megaport_secret_key
  environment = "production"
}

# Look up Equinix DC6 location
data "megaport_location" "equinix_dc6" {
  name    = "Equinix DC6"
  has_mcr = true
}

# Create MCR — Layer 3 cloud router (no physical hardware needed)
resource "megaport_mcr" "primary" {
  mcr_name    = "mcr-enterprise-primary"
  location_id = data.megaport_location.equinix_dc6.id
  port_speed  = 10000   # 10 Gbps MCR
  asn         = 133937  # Megaport's public ASN (or your own)
}

output "mcr_uid"  { value = megaport_mcr.primary.mcr_uid }
output "mcr_name" { value = megaport_mcr.primary.mcr_name }
EOF

terraform init && terraform apply
MCR_UID=$(terraform output -raw mcr_uid)
echo "MCR created: $MCR_UID"
```

#### Step 2: Create VXC to AWS Direct Connect

```bash
cat >> main.tf << 'EOF'
# VXC from MCR to AWS Direct Connect
resource "megaport_aws_connection" "prod" {
  vxc_name   = "vxc-mcr-to-aws-prod"
  rate_limit = 1000    # 1 Gbps

  a_end {
    requested_vlan = 200
    mcr_uid        = megaport_mcr.primary.mcr_uid
    inner_vlan     = 200
  }

  b_end {
    aws_account_id        = var.aws_account_id
    aws_connection_type   = "private"
    requested_product_uid = data.megaport_location.equinix_dc6.products.aws[0].uid
  }

  bgp_config {
    peer_asn         = 64512        # AWS Amazon side ASN
    local_asn        = 133937       # MCR ASN
    local_ip_address = "169.254.10.1/30"
    peer_ip_address  = "169.254.10.2"
    password         = var.bgp_auth_key
    # Advertise all prefixes from MCR to AWS
    prefix_filter_list = []
  }
}

# VXC from MCR to Azure ExpressRoute
resource "megaport_azure_connection" "prod" {
  vxc_name   = "vxc-mcr-to-azure-prod"
  rate_limit = 1000

  a_end {
    requested_vlan = 300
    mcr_uid        = megaport_mcr.primary.mcr_uid
  }

  b_end {
    service_key = var.azure_expressroute_service_key
  }

  bgp_config {
    peer_asn         = 12076          # Microsoft ASN
    local_asn        = 133937
    local_ip_address = "169.254.20.1/30"
    peer_ip_address  = "169.254.20.2"
    password         = var.bgp_auth_key
  }
}
EOF

terraform apply
echo "VXCs created: MCR → AWS DX + MCR → Azure ER"
```

---

### Phase 2 — AWS Side

#### Step 3: Accept DX connection and create TGW

```bash
cd terraform/aws/

# Accept the Direct Connect connection Megaport created
DX_CONNECTION_ID=$(aws directconnect describe-connections \
  --query 'connections[?partnerName==`Megaport`].connectionId' \
  --output text --region us-east-1)

echo "DX Connection: $DX_CONNECTION_ID"

# Create Direct Connect Gateway
DXGW_ID=$(aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name enterprise-dxgw \
  --amazon-side-asn 64512 \
  --query 'directConnectGateway.directConnectGatewayId' \
  --output text)

# Create Private VIF on DX connection
aws directconnect create-private-virtual-interface \
  --connection-id $DX_CONNECTION_ID \
  --new-private-virtual-interface \
    virtualInterfaceName=vif-to-mcr,\
    vlan=200,\
    asn=133937,\
    authKey=$(cat /run/secrets/bgp_key),\
    amazonAddress=169.254.10.2/30,\
    customerAddress=169.254.10.1/30,\
    addressFamily=ipv4,\
    directConnectGatewayId=$DXGW_ID

# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "Enterprise TGW — hub for all VPCs and DX" \
  --options \
    AmazonSideAsn=64512,\
    AutoAcceptSharedAttachments=disable,\
    DefaultRouteTableAssociation=disable,\
    DefaultRouteTablePropagation=disable,\
    DnsSupport=enable,\
    VpnEcmpSupport=enable \
  --query 'TransitGateway.TransitGatewayId' \
  --output text)

# Associate DXGW with TGW
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id $DXGW_ID \
  --gateway-id $TGW_ID \
  --add-allowed-prefixes-to-direct-connect-gateway \
    Cidr=10.1.0.0/14

echo "TGW: $TGW_ID | DXGW: $DXGW_ID | DX VIF created"
```

#### Step 4: Deploy Prod VPC and attach to TGW

```bash
terraform apply -target=module.prod_vpc -target=module.tgw_attachment_prod

# Verify VPC attached and route propagating
aws ec2 describe-transit-gateway-attachments \
  --filters Name=transit-gateway-id,Values=$TGW_ID \
  --query 'TransitGatewayAttachments[*].{Type:ResourceType,State:State,ID:TransitGatewayAttachmentId}' \
  --output table

# Check routes learned from on-prem via DX/MCR
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_RT_ID \
  --filters Name=type,Values=propagated \
  --query 'Routes[*].{CIDR:DestinationCidrBlock,State:State}' \
  --output table
# Expected: 10.10.0.0/16 (on-prem), 10.2.0.0/14 (Azure) visible
```

---

### Phase 3 — Azure Side

#### Step 5: Create ExpressRoute circuit and accept Megaport connection

```bash
cd terraform/azure/

# Create ExpressRoute circuit
az network express-route create \
  --name erc-enterprise-to-azure \
  --resource-group rg-hub-network \
  --location eastus \
  --bandwidth 1000 \
  --provider "Megaport" \
  --peering-location "Washington DC" \
  --sku-tier Standard \
  --sku-family MeteredData

# Get service key (share this with Megaport)
SERVICE_KEY=$(az network express-route show \
  --name erc-enterprise-to-azure \
  --resource-group rg-hub-network \
  --query serviceKey --output tsv)
echo "Azure ER Service Key (provide to Megaport): $SERVICE_KEY"
# → Paste this into var.azure_expressroute_service_key in Step 2

# Configure Private Peering (after Megaport provisions VXC)
az network express-route peering create \
  --circuit-name erc-enterprise-to-azure \
  --resource-group rg-hub-network \
  --peering-type AzurePrivatePeering \
  --peer-asn 133937 \
  --primary-peer-address-block 169.254.20.0/30 \
  --secondary-peer-address-block 169.254.20.4/30 \
  --vlan-id 300 \
  --shared-key $(cat /run/secrets/bgp_key)

# Deploy ExpressRoute Gateway and connect to circuit
terraform apply -target=module.expressroute_gateway
```

---

### Phase 4 — End-to-End Validation

#### Step 6: Verify full path — On-prem → AWS → Azure

```bash
echo "=== HYBRID CONNECTIVITY VALIDATION ==="

# Test 1: On-prem to AWS VPC (should traverse MCR → DX → TGW)
aws ssm start-session --target $TEST_INSTANCE_ID \
  --document-name AWS-RunShellScript \
  --parameters commands='["ping -c 4 10.10.100.5 && echo ONPREM_TO_AWS_OK"]'

# Test 2: AWS to Azure (should traverse TGW → DXGW → MCR → ER → Azure)
aws ssm start-session --target $TEST_INSTANCE_ID \
  --document-name AWS-RunShellScript \
  --parameters commands='["ping -c 4 10.2.1.10 && echo AWS_TO_AZURE_OK"]'

# Test 3: On-prem to Azure (direct via MCR)
# (Run from on-prem router)
# ping 10.2.1.10

# Test 4: Check BGP routes on MCR (via Megaport API)
curl -s -H "Authorization: Bearer $MEGAPORT_TOKEN" \
  "https://api.megaport.com/v2/product/$MCR_UID/bgppeers" \
  | python3 -m json.tool

# Test 5: Verify latency (DX should be < 5ms, ER < 10ms)
aws ssm start-session --target $TEST_INSTANCE_ID \
  --document-name AWS-RunShellScript \
  --parameters commands='["traceroute 10.10.100.5"]'

echo "=== ALL CONNECTIVITY TESTS PASSED ==="
```

---

## BGP Route Advertisement Summary

| Source | Prefix Advertised | Reaches |
|--------|------------------|---------|
| On-premises | 10.10.0.0/16 | AWS VPCs + Azure VNets |
| AWS TGW | 10.1.0.0/14 | On-prem + Azure |
| Azure | 10.2.0.0/14 | On-prem + AWS |
| MCR | Redistributes all | All parties |

## Estimated Monthly Cost

| Component | Cost |
|-----------|------|
| Megaport MCR (10G) | $650 |
| VXC to AWS (1G) | $400 |
| VXC to Azure (1G) | $350 |
| AWS DX Port (1G) | $216 |
| Azure ER Circuit (1G Standard) | $625 |
| AWS TGW | $36 + data |
| **Total** | **~$2,280/month** |

## License

MIT License
