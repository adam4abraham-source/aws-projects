# Lab 6: Highly Available Web Tier with ALB + Auto Scaling

A two-AZ web tier on AWS: an internet-facing Application Load Balancer in public subnets, distributing traffic to an Auto Scaling Group of EC2 instances that live in **private** subnets. Built by hand in the console on top of the existing `lab3-vpc`.

## Architecture

```
Internet
   |
   v
Application Load Balancer (lab6-alb)        public subnets: us-east-2a / us-east-2b
   |  HTTP :80  (lab6-alb-sg: 0.0.0.0/0)
   v
Target Group (lab6-tg)  -- health check: HTTP /
   |  HTTP :80  (lab6-app-sg: source = lab6-alb-sg only)
   v
Auto Scaling Group (lab6-asg)               private subnets: 10.2.11.0/24 (2a), 10.2.12.0/24 (2b)
   min 2 / desired 2 / max 4
   |
   v  (outbound only, to install packages)
NAT Gateway (lab6-nat) in public subnet 2a  ->  Internet Gateway
```

## Resources

| Resource | Name | Notes |
|---|---|---|
| VPC | `lab3-vpc` (10.2.0.0/16, us-east-2) | Reused from the previous lab |
| Security group | `lab6-alb-sg` | Inbound HTTP 80 from the internet |
| Security group | `lab6-app-sg` | Inbound HTTP 80 **only from `lab6-alb-sg`** (SG-to-SG reference) |
| Launch template | `lab6-template` | Amazon Linux 2023, t3.micro, user data installs Apache |
| NAT Gateway | `lab6-nat` | Zonal, public subnet 2a, so private instances can reach package repos |
| Target group | `lab6-tg` | Instance targets, HTTP 80, health check on `/` |
| Load balancer | `lab6-alb` | Internet-facing, IPv4, subnets in 2a and 2b |
| Auto Scaling group | `lab6-asg` | Private subnets, min 2 / desired 2 / max 4, ELB health checks, 300 s grace period |

### User data

Each instance installs Apache and writes a page showing its own instance ID and Availability Zone (read from IMDSv2), which makes load balancing visible:

```bash
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>lab6 server: $ID ($AZ)</h1>" > /var/www/html/index.html
systemctl enable --now httpd
```

## Build steps

1. Create `lab6-alb-sg` and `lab6-app-sg` in `lab3-vpc`.
2. Create the launch template `lab6-template` (no subnet; the ASG chooses it).
3. Create the NAT Gateway in a public subnet and point `0.0.0.0/0` in the private route table at it.
4. Create the target group `lab6-tg` with no registered targets.
5. Create the ALB across the two public subnets and forward HTTP:80 to `lab6-tg`.
6. Create the ASG in the two private subnets, attach it to `lab6-tg`, enable ELB health checks, set 2 / 2 / 4.

## Results

- Opening the ALB DNS name returned a page from an instance in the private subnets; refreshing showed responses from instances in **us-east-2a** and **us-east-2b**.
- The instances have **no public IP** and are reachable only through the ALB.
- **Self-healing test:** I terminated one instance manually. The ASG detected capacity below the minimum and launched a replacement in about a minute, while the site stayed up through the other instance.

Screenshots: `alb-server-2a.png`, `alb-server-2b.png`, `instances-replaced.png`, `asg-activity.png`.

## What went wrong and what I learned

- **Stale NAT route.** The private route table still had a `0.0.0.0/0` route pointing to a NAT Gateway I had deleted after the previous lab (status **Blackhole**). I edited the route to target the new NAT instead of adding a second one.
- **Elastic IP would not release** while still associated with the NAT Gateway. The NAT has to be deleted first, then the console view refreshed, then the address released.
- **Why SG-to-SG rules:** referencing `lab6-alb-sg` as the source means only the load balancer can reach the app tier, with no IP ranges to maintain.
- **Private subnets need a NAT** only for outbound traffic such as package installs; inbound traffic arrives through the ALB.

## Cleanup (order matters)

1. Delete the Auto Scaling group (terminates the instances).
2. Delete the load balancer, then the target group.
3. Delete the NAT Gateway, wait until it shows **Deleted**, then release the Elastic IP.
4. Delete the launch template.
5. Delete `lab6-app-sg`, then `lab6-alb-sg`.

The ALB and NAT Gateway are billed per hour, so this lab was built and deleted in a single session. `lab3-vpc`, its subnets and route tables were kept for the next labs.

## Next

Lambda with an S3 trigger.
