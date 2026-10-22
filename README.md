<img width="1536" height="1024" alt="vpc-peering-diagram" src="https://github.com/user-attachments/assets/6b6309a5-883d-469f-91c6-596521ba645a" />
# AWS VPC Peering Project with Bastion Host

A hands-on AWS networking project demonstrating secure, private communication between EC2 instances in two separate Virtual Private Clouds (VPCs). The architecture uses a public bastion host, private subnets, custom route tables, security groups, and VPC Peering.

## Project Overview

The objective was to deploy three Ubuntu EC2 instances across two non-overlapping VPCs, provide controlled SSH access to the private instances, and establish bidirectional private connectivity between EC2-B and EC2-C without assigning either a public IPv4 address.

**AWS Region:** Asia Pacific (Mumbai) — `ap-south-1`

## Architecture

```text
                         MacBook (SSH client)
                                  |
                         SSH from allowed IP
                                  |
                    +-------------v--------------+
                    | VPC A: 10.0.0.0/16          |
                    |                            |
                    | Public-Subnet-A            |
                    | 10.0.1.0/24                 |
                    | EC2-A: Public Bastion Host  |
                    |       |                    |
                    |       | Private SSH        |
                    |       v                    |
                    | Private-Subnet-A           |
                    | 10.0.2.0/24                 |
                    | EC2-B: Private Instance     |
                    +-------------+--------------+
                                  |
                         VPC Peering (Active)
                                  |
                    +-------------+--------------+
                    | VPC B: 10.200.0.0/16        |
                    |                            |
                    | Private-Subnet-B           |
                    | 10.200.1.0/24               |
                    | EC2-C: Private Instance     |
                    +----------------------------+

       EC2-B <---------- Private ICMP ----------> EC2-C
```

> Add your architecture screenshot later at `architecture/vpc-peering-diagram.png`, then replace or supplement the text diagram with `![AWS VPC Peering architecture](architecture/vpc-peering-diagram.png)`.

## AWS Services and Tools

- Amazon VPC and subnets
- Amazon EC2 (Ubuntu Server 24.04 LTS)
- VPC Peering
- Internet Gateway
- Custom route tables
- Security groups
- SSH and ProxyJump
- ICMP (`ping`) for connectivity testing

## Network Configuration

| Resource | CIDR | Purpose |
| --- | --- | --- |
| Project-VPC-A | `10.0.0.0/16` | Bastion and first private instance |
| Public-Subnet-A | `10.0.1.0/24` | EC2-A (bastion host) |
| Private-Subnet-A | `10.0.2.0/24` | EC2-B |
| Project-VPC-B | `10.200.0.0/16` | Second private instance |
| Private-Subnet-B | `10.200.1.0/24` | EC2-C |

### EC2 Instances

| Instance | Network | Public IPv4 | Role |
| --- | --- | --- | --- |
| Project-EC2-A-Public | Public-Subnet-A | Enabled | SSH bastion |
| Project-EC2-B-Private | Private-Subnet-A | Disabled | Private server in VPC A |
| Project-EC2-C-Private | Private-Subnet-B | Disabled | Private server in VPC B |

## Routing Configuration

| Route table | Destination | Target |
| --- | --- | --- |
| Public-RT-A | `10.0.0.0/16` | local |
| Public-RT-A | `0.0.0.0/0` | Project-IGW-A |
| Private-RT-A | `10.0.0.0/16` | local |
| Private-RT-A | `10.200.0.0/16` | Active VPC Peering Connection |
| Private-RT-B | `10.200.0.0/16` | local |
| Private-RT-B | `10.0.0.0/16` | Active VPC Peering Connection |

The private subnet route tables have no default route to the Internet Gateway. Both sides require a route to the remote VPC for bidirectional communication.

## Security Group Rules

| Security group | Key inbound rules |
| --- | --- |
| SG-Public-A | SSH (TCP 22) from the administrator's IP (`/32`) |
| SG-Private-A | SSH (TCP 22) from SG-Public-A; ICMP from `10.200.1.0/24` |
| SG-Private-B | SSH (TCP 22) and ICMP from `10.0.2.0/24` |

The project used default outbound allow-all rules. Security groups are stateful; response traffic for permitted connections is automatically allowed.

## Implementation Steps

1. Created two VPCs with non-overlapping private CIDR ranges.
2. Created one public subnet and two private subnets.
3. Created and attached an Internet Gateway to VPC A.
4. Created three custom route tables and associated them with their respective subnets.
5. Configured security groups for restricted SSH and cross-VPC ICMP access.
6. Launched EC2-A, EC2-B, and EC2-C with the same SSH key pair.
7. Configured SSH ProxyJump on the local MacBook to access the private instances without copying the private key onto EC2 hosts.
8. Created and accepted the VPC Peering Connection between VPC A and VPC B.
9. Added peering routes to both private route tables.
10. Tested SSH and ICMP connectivity.

## SSH Access

The local SSH configuration uses host aliases and `ProxyJump`:

```sshconfig
Host ec2-a
    HostName <EC2_A_PUBLIC_IP>
    User ubuntu
    IdentityFile ~/.ssh/<YOUR_PRIVATE_KEY_FILE>.pem
    IdentitiesOnly yes

Host ec2-b
    HostName <EC2_B_PRIVATE_IP>
    User ubuntu
    IdentityFile ~/.ssh/<YOUR_PRIVATE_KEY_FILE>.pem
    IdentitiesOnly yes
    ProxyJump ec2-a

Host ec2-c
    HostName <EC2_C_PRIVATE_IP>
    User ubuntu
    IdentityFile ~/.ssh/<YOUR_PRIVATE_KEY_FILE>.pem
    IdentitiesOnly yes
    ProxyJump ec2-a,ec2-b
```

Replace placeholders with your actual values **only in your local SSH config**. Do not commit private keys or sensitive connection details to GitHub.

```bash
ssh ec2-b
ssh ec2-c
```

## Connectivity Verification

The following tests were completed successfully:

| Test | Result |
| --- | --- |
| MacBook to EC2-B via SSH ProxyJump | Passed |
| MacBook to EC2-C via SSH ProxyJump | Passed |
| EC2-B to EC2-C (`ping -c 4`) | Passed — 0% packet loss |
| EC2-C to EC2-B (`ping -c 4`) | Passed — 0% packet loss |
| VPC Peering status | Active |

Example commands, using the destination's actual private IPv4 address:

```bash
# Run on EC2-B
ping -c 4 <EC2_C_PRIVATE_IP>

# Run on EC2-C
ping -c 4 <EC2_B_PRIVATE_IP>
```

The tests verify private connectivity for the tested SSH and ICMP traffic; they do not imply that all ports or protocols are permitted.

## Screenshots

Add sanitized evidence under `screenshots/`:

- VPCs and subnets
- Route table entries
- Active VPC Peering Connection
- EC2 instance overview
- Successful ping from EC2-B to EC2-C
- Successful ping from EC2-C to EC2-B

Remove or obscure account IDs, public IPs where unnecessary, and other sensitive information before publishing. **Never upload `.pem` files, AWS access keys, credentials, or secrets.**

## Key Learnings

- VPC Peering requires non-overlapping CIDR ranges.
- An Active peering connection alone is insufficient: routes and security rules must also permit traffic.
- VPC Peering does not support transitive routing or sharing a peer's Internet Gateway.
- A bastion host and SSH ProxyJump can provide controlled access to private instances without storing the private key on intermediate servers.

## Cost Awareness

Review EC2, EBS, public IPv4, and VPC Peering data transfer charges in your AWS account. Stop or remove resources when they are no longer needed, taking care to preserve project evidence first.
