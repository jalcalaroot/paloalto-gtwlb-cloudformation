# Palo Alto VM-Series Security VPC with Gateway Load Balancer (CloudFormation)

CloudFormation template that deploys a two-AZ "security VPC" with two Palo Alto Networks VM-Series firewalls behind an AWS Gateway Load Balancer (GWLB), GWLB endpoints, NAT gateways and an optional Transit Gateway attachment for centralized traffic inspection. (`gtwlb` in the repo name and `tamplate` in the file name are typos for GWLB / template.)

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `paloalto-gtwlb-tamplate.yaml`

```
 spoke VPCs --TGW--> TGW subnets --0.0.0.0/0--> GWLB endpoints --> GWLB (GENEVE 6081) --> VM-Series x2 (data ENI)
                                                       |                     
                                     GWLBE subnets --0.0.0.0/0--> NAT GW --> IGW
                     NAT GW subnets: RFC1918 return traffic --> GWLB endpoints
```

- Security VPC with IGW and four subnet tiers in each of 2 AZs: NAT GW, GWLB endpoint (GWLBE), TGW, VM-Series data
- Two NAT gateways with EIPs; route tables send RFC1918 (`10/8`, `172.16/12`, `192.168/16`) through the GWLB endpoints and, when a TGW is given, back to the TGW
- Gateway Load Balancer (cross-zone enabled), GENEVE target group (TCP health check on port 80) and listener
- VPC endpoint service for the GWLB plus one GWLB endpoint per AZ
- Lambda-backed custom resource (Python 3.13, inline code) that looks up the endpoint service name
- Two VM-Series instances (BYOL-style user data with GWLB inspection enabled, or S3 bootstrap), each with a data ENI and a management ENI in the NAT GW (public) subnet with an EIP
- Optional: TGW VPC attachment, S3 bootstrap IAM role/instance profile and S3 gateway endpoints restricted to the bootstrap bucket
- Organisation-style tags (`workload:*`, `security:*`, ...) on most resources
- Output: `GWLBServiceId` (the endpoint service name, for extra GWLB endpoints in spoke VPCs)

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `RemoteManagementCIDR` | CIDR allowed to SSH/HTTPS to the firewall management | `0.0.0.0/0` |
| `ManagementPrefixList` | Prefix list also allowed to management | (required) |
| `SecurityVPCCIDR` | Security VPC CIDR | `10.0.0.0/24` |
| `SecurityVPC*SubnetCIDRAZ1/AZ2` | `/28` CIDRs for NAT GW, GWLBE, TGW and data subnets | `10.0.0.0/28` ... `10.0.0.112/28` |
| `VMSeriesInstanceType` | Firewall instance type | `m5.xlarge` |
| `VMSeriesAMI` | VM-Series AMI ID (PAN-OS 10.0.2+) | `<VMSERIES_AMI_ID>` |
| `EC2KeyPair` | Key pair for firewall admin access | `<KEY_PAIR_NAME>` |
| `TGWID` | Transit Gateway to attach (blank = none) | empty |
| `VmseriesBootstrapS3BucketName` | Bootstrap bucket (blank = no bootstrap) | empty |
| `pTagValueTechOwner` / `pTagValueBussOwner` | Owner email tags | `<OWNER_EMAIL>` |
| `pTagValueCostCenter` | Cost center tag | `<COST_CENTER>` |
| other `pTagValue*` | Environment, project, criticity, etc. tags | various |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file paloalto-gtwlb-tamplate.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    VMSeriesAMI=<VMSERIES_AMI_ID> EC2KeyPair=<KEY_PAIR_NAME> \
    RemoteManagementCIDR=<ADMIN_CIDR> ManagementPrefixList=<PREFIX_LIST_ID> \
    TGWID=<TGW_ID> pTagValueTechOwner=<OWNER_EMAIL> pTagValueBussOwner=<OWNER_EMAIL>
```

Subscribe to the VM-Series AMI in AWS Marketplace first. Enable appliance mode on the TGW attachment after creation, and add TGW route tables for the spoke VPCs yourself. The custom resource sleeps 10 minutes, so stack creation is slow.

## Security notes

- `RemoteManagementCIDR` defaults to `0.0.0.0/0`: SSH and HTTPS to the firewall management interfaces (with public EIPs) are open to the internet unless overridden.
- The data security group allows all traffic from `0.0.0.0/0` (common for GWLB GENEVE, but scope it to the VPC CIDR if possible).
- Firewall root volumes are gp2 and not encrypted; detailed monitoring is off.
- The bootstrap role grants `cloudwatch:*` on `*`.
- The custom-resource Lambda only answers `Create` and `Delete`; an `Update` event never gets a response and the stack update would hang until timeout.
- Firewalls start unconfigured unless a bootstrap bucket is provided; harden the default admin access before exposing them.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

Remove any GWLB endpoints created in other VPCs from this endpoint service, and TGW routes pointing at the attachment, before deleting. The bootstrap bucket is not managed by this stack.

## License

MIT — see [LICENSE](LICENSE).
