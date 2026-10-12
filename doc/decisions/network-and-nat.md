# Network: public subnet vs NAT gateway vs VPC endpoints

Status: open. The security team decides (Gunnar, 2026-10-13). This file is the input for that
decision, carried over from TaskFest; nothing is decided for this project yet.

## Context

TaskFest today: Fargate tasks in **public subnets with a public IP** and a security group that
admits only the load balancer. **No NAT gateway.** RDS and the internal load balancer are in
private subnets with no routes. TaskFest's reasoning is in its
`doc/decisions/deployment-and-aws.md` (§ "Fargate tasks in public subnets, with no NAT gateway").

**VPC endpoints alone aren't an option if the backend calls anything outside AWS.** TaskFest's
backend calls `accounts.google.com` (exchanging the sign-in code for a token) and `gravatar.com`.
Endpoints only reach AWS services, so they add to a NAT gateway or a public IP; they don't
replace them. Whether this project calls outside AWS is still to be established.

## Options

Prices are Frankfurt (eu-central-1). Figures marked **(unverified)** rest on the NAT gateway
price below.

| | Public subnet (TaskFest now) | Private + NAT gateway | Private + VPC endpoints |
|---|---|---|---|
| Fixed cost / month | $3.65 (task's public IPv4) | ~$38 NAT **(unverified)** + $3.65 Elastic IP, minus the task's IP: **~+$38 (unverified)**; ~+$76 with one per AZ **(unverified)** | $8.76 per endpoint per AZ; ECR ×2, Logs, Secrets Manager = **$35** (1 AZ) / $70 (2 AZ), plus NAT or public IP anyway |
| Per GB | – | $0.052 **(unverified)** | $0.01 |
| Inbound | security group only | no route from the internet, plus the security group | same as NAT |
| Outbound | open (Trivy AWS-0104 suppressed) | open unless filtered (DNS Firewall; Network Firewall ~$290/month) | only to AWS, restrictable by endpoint policies; can't reach Google etc. |
| Outgoing IP | changes per deploy | fixed, can be allow-listed | – |
| Complexity | lowest | moderate: NAT, Elastic IP, private route tables, tasks moved to private subnets | highest: one endpoint and DNS per service; a missing one is a timeout |
| AWS best practice | allowed, not recommended for production | the standard setup | strictest, usually combined with NAT |

- **The free S3 gateway endpoint** goes with any private setup: image layers and S3 traffic stay
  off the NAT's per-GB charge.
- **Cheaper, not checked:** a NAT instance (e.g. fck-nat on `t4g.nano`, ~$4–8/month, but a server
  to patch and a single point of failure); IPv6 with a free egress-only internet gateway, if every
  outside service is reachable over IPv6.

## Recommendation (from the TaskFest session)

Public subnet while it's a demo. Once it's more than that: private subnets, one NAT gateway
(behind an on/off switch like TaskFest's `running`), plus the S3 gateway endpoint. Interface
endpoints only to restrict where data can go.

## Prices and how sure they are

- **Verified:** endpoint $0.012/h per AZ + $0.01/GB, and public IPv4 $0.005/h, from AWS's price
  list for eu-central-1.
- **Unverified:** NAT gateway $0.052/h + $0.052/GB in Frankfurt, from the author's knowledge and
  not checked against the price list. TaskFest's docs say "about $33/month", which is the US East
  rate. Every NAT figure above and below depends on this; check it against the AWS price list
  before relying on it.
- For comparison, TaskFest qa runs at about $56/month (~$1.86/day) on Graviton, measured; a NAT
  gateway would make that about $94 **(unverified, depends on the NAT price)**.

## Where to look in the TaskFest repo

- `deployment/aws-tofu/modules/environment/network.tf`, `security.tf`, `ecs.tf`: the VPC, the
  security groups, Fargate with the ECR pull-through cache.
- `doc/decisions/deployment-and-aws.md`: why each AWS choice was made.
- `doc/deployment/cost.md`: measured costs.
