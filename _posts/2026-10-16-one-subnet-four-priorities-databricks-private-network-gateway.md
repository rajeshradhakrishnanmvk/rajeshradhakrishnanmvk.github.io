---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - networking
---

# One Subnet, Four Priorities: What Databricks Private Network Gateway Changes for an Architect

On 14 October 2026, Alex Esibov, Manish Bansal, Stanley Wang, Justin Ly, and Vijay Raja published Private Network Gateway. Serverless compute gets one managed path into a subnet the customer delegates. Routes, DNS, and firewalls past that subnet decide what is reachable. The post calls Azure a public preview and AWS a private preview, and it says Google Cloud is in development. The pages I would take into a review do not share one label.

The Azure overview and configure article both say Public Preview, switched on from the account console. The AWS overview, last updated 3 September, still says Private Preview and is unlisted. The AWS configure article, updated 13 October, is unlisted too. The matching Google Cloud URL returned 404. The 17 June summit post announced a private preview on Azure only, and the 3 September Azure notes still say Beta. The October product notes I read, on AWS and on Azure, have no gateway bullet. A lag warning on those notes is not a schedule for a feature they do not list.

## The subnet is the attachment

Classic compute already sits in the customer's virtual network, so peering reaches the data. Serverless compute sits in a Databricks-managed network, and a private database or an internal API needs a path someone builds. The October post describes today's path as a private endpoint, a load balancer, and a proxy virtual machine per resource. Serverless Private Link stays the path for a cloud-managed service such as Amazon S3 or Azure Data Lake Storage. The gateway is the other path: one object inside the network connectivity configuration the account already has. Attach that configuration to workspaces in the same region, and serverless products there can use it.

Azure injects the gateway into a subnet delegated to `Microsoft.Databricks/workspaces`. AWS places a network interface in a subnet the customer provides. Databricks assumes a role in that account. The trust principal is `png-management-role` in AWS account `414351767826`, and the external id is the customer's Databricks account id. The same policy allows VPC Lattice resource-gateway calls. The overview says interface. The policy describes a second construct. I would keep both.

The October post says the path is dedicated and that workspace controls run before traffic leaves. The docs do not show a test of that isolation.

Outbound traffic takes the first match. Both overviews print the same four rows. The AWS page still includes the Azure service-endpoint cell. I would use the AWS configure page for what an AWS gateway can reach.

| Order | Path | What misses the gateway |
| --- | --- | --- |
| 1 | Private endpoint rule | That resource, even in `ALL_TRAFFIC` |
| 2 | S3 and DynamoDB gateway endpoints; Azure service endpoints | Those names. Azure Blob storage always takes this path. |
| 3 | Private network gateway | Listed DNS names, or whatever the rows above left |
| 4 | Default serverless egress | Every other name |

## Two modes, two allow lists

`SPECIFIC_DESTINATIONS` sends the names on the gateway and leaves every other name alone. The docs recommend it. A name matches names under it, so `mydb.contoso.com` matches `sub.mydb.contoso.com`. Egress control allows those destinations without a second entry. `ALL_TRAFFIC` sends only the remainder, and each destination has to be added to the egress policy by hand.

```json
{
  "traffic_mode": "SPECIFIC_DESTINATIONS",
  "destinations": [
    { "destination_type": "DNS_NAME", "value": "payments-db.internal.contoso.com" }
  ]
}
```

The firewall behind the subnet does not see Private Link or the backbone path. On Azure, a service-endpoint suffix takes that suffix path. The troubleshooting table names Azure SQL Database and Key Vault, and it says a private-only resource needs a more specific `DNS_NAME`. An egress-gateway deny is evaluated before this gateway. Inspection starts at row 3.

The October post says more than 75 percent of customers interviewed blamed network changes for workloads that stayed on classic compute. It gives no count and no method. [The Clarity](https://theclarity.today/story/databricks-gateway-gives-serverless-one-path-per-network-535ad02b), updated 14 October, marks that claim insufficient. I would leave the percentage in the blog. Fabiano Seki of Banco Bradesco is quoted there on reusing a workspace subnet. The Azure configure page wants a subnet no other resource uses, `/28` or larger. I would follow the configure page.

<img src="/assets/images/private-network-gateway-four-priorities.svg" alt="Four ordered paths for serverless egress: Private Link, cloud backbone endpoints, the private network gateway, and default egress" title="The subnet attaches the gateway. Private Link and backbone endpoints still win, including in ALL_TRAFFIC." style="width:100%; height:auto;">

## What each cloud will let me build

| | Azure | AWS |
| --- | --- | --- |
| Label | Public Preview, Premium. Billing starts on or after 1 December 2026. What's coming prints no unit. The add-ons page I opened has no gateway line. | Private Preview, Enterprise. The 3 September overview says it is not billed in preview. Regions are unpublished. Ask the account team for the zone. |
| Create | Console, Networking v1 API, or Terraform. Bandwidth is 10, 30, 50, or 100 Gbps. The console starts at 30. The API requires the field. Terraform says there is no server-side default. A change confirms the request and does not say when capacity is present. | REST API only. The configure page says no UI and no Terraform in preview. The provider resource says Azure only, and it still documents `aws_cloud_connection`. I would not apply that block. |
| Subnet | Unused, delegated to `Microsoft.Databricks/workspaces`, `/28` or larger, same region as the NCC, same Entra tenant. Twenty-five regions are named, including `centralindia`. | One subnet, one Availability Zone, `/28` as the practical minimum, plus a security group. The page recommends no inbound rules and outbound rules limited to destination ports. |

Google Cloud stays off the table until a docs page exists. An NCC holds at most two gateways. A gateway holds at most two resolvers and 100 destinations. The limitations line I would quote is serverless runtime products and AI services. The blog also names Lakeflow Connect, a private MCP server, a private package index, and a stable source address. That list is not the limitations sentence. `FAILED` is terminal: delete it, fix the region or the delegation, and create another. A new subnet is a new gateway. Delete cuts the path and returns no operation to poll.

## Where it fits, and where it does not

I would use it on Azure, in a named region, when several private names sit behind one VNet and the job is already serverless. Ship `SPECIFIC_DESTINATIONS` first, on an empty subnet, with the gateway `ESTABLISHED`, after the name is shown not to match a service-endpoint suffix. Use `ALL_TRAFFIC` when a firewall must see what rows 1 and 2 left, and write Blob, Private Link, and suffix exceptions beside it. Keep classic compute until that path is proven.

I would leave AWS until the account team names the zone, and I would leave Google Cloud until a page exists. S3, DynamoDB, Azure Blob, and any resource that already has a private endpoint rule stay on those paths. A caller the limitations line does not name, a workspace in another region, a shared subnet, and a made-up rate stay out.

## What goes in the design doc

Write the label, Azure Public Preview or AWS Private Preview, and whether enablement is the previews page or the account team. Draw the four priorities before the mode. Pin a dedicated `/28`, the region, two gateways, 100 names, and two resolvers. Put the mode next to the egress policy, and leave the price blank. The subnet is how serverless enters the network. The first match is often a path the gateway does not own.

## References

- Alex Esibov, Manish Bansal, Stanley Wang, Justin Ly, and Vijay Raja, "Introducing Private Network Gateway," Databricks Blog, 14 October 2026. [https://www.databricks.com/blog/introducing-private-network-gateway](https://www.databricks.com/blog/introducing-private-network-gateway)
- Jason Wu, Samrat Ray, Filippo Seracini, Alex Esibov, Vijay Raja, Kelly Albano, Robert Zhang, and Mia Penfold Lopez, "What's new in Databricks Data + AI Platform security and compliance at Data + AI Summit 2026," Databricks Blog, 17 June 2026. [https://www.databricks.com/blog/whats-new-databricks-platform-security-and-compliance-data-ai-summit-2026](https://www.databricks.com/blog/whats-new-databricks-platform-security-and-compliance-data-ai-summit-2026)
- Microsoft, "Private network gateway" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/private-network-gateway](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/private-network-gateway)
- Microsoft, "Configure a private network gateway" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/private-network-gateway/configure-private-network-gateway](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/private-network-gateway/configure-private-network-gateway)
- Microsoft, "What's coming?", billing for private network gateway. [https://learn.microsoft.com/en-us/azure/databricks/release-notes/whats-coming](https://learn.microsoft.com/en-us/azure/databricks/release-notes/whats-coming)
- Microsoft, "September 2026" release notes, private network gateway in Beta on 3 September 2026. [https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/september](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/september)
- Microsoft, "October 2026" release notes (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october)
- Databricks, "October 2026" release notes (AWS). [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Databricks, "Private network gateway" (AWS), last updated 3 September 2026. [https://docs.databricks.com/aws/en/security/network/serverless-network-security/private-network-gateway](https://docs.databricks.com/aws/en/security/network/serverless-network-security/private-network-gateway)
- Databricks, "Configure a private network gateway" (AWS), last updated 13 October 2026. [https://docs.databricks.com/aws/en/security/network/serverless-network-security/private-network-gateway/configure-private-network-gateway](https://docs.databricks.com/aws/en/security/network/serverless-network-security/private-network-gateway/configure-private-network-gateway)
- Databricks, "Data + AI Platform and Add-Ons" pricing page, checked 16 October 2026. [https://www.databricks.com/product/pricing/platform-addons](https://www.databricks.com/product/pricing/platform-addons)
- Databricks Terraform provider, `databricks_private_network_gateway`. [https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/private_network_gateway](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/private_network_gateway)
- The Clarity, "Databricks gateway gives serverless one path per network," updated 14 October 2026. [https://theclarity.today/story/databricks-gateway-gives-serverless-one-path-per-network-535ad02b](https://theclarity.today/story/databricks-gateway-gives-serverless-one-path-per-network-535ad02b)
