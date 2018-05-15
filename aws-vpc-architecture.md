# AWS VPC Architecture
This documents our AWS Virtual Private Cloud networking infrastructure.

## Overview
We have four VPCs. Three are in the region us-west-2 (Oregon) and the fourth is
in the region us-west-1 (Northern California).

Each VPC has a CIDR block with a subnet mask of `255.255.0.0` or `/16`. Each
VPC has one subnet per availability zone with a CIDR block of `255.255.255.0`
or `/24`. VPC peering connections are established between certain VPCs to allow
for private inter-VPC routing. All peering connections have bidirectional
routing.

All VPCs have an internet gateway allowing the ec2 servers access to the
internet.

## web-services
The web-services VPC is for general web services that support our
infrastructure but are not factom nodes. All other VPCs have peering
connections with this VPC.

### Details
| Key     | Value                   |
|---------|-------------------------|
| Name    | web-services            |
| VPC ID  | `vpc-0e6a18ebd9ff59153` |
| Region  | us-west-2 (Oregon)      |
| CIDR    | `10.20.0.0/16`          |

### Subnets
| Availability Zone | CIDR           | Subnet ID                  |
|-------------------|----------------|----------------------------|
| us-west-2a        | `10.20.0.0/24` | `subnet-0e6bb4aab86249651` |
| us-west-2b        | `10.20.1.0/24` | `subnet-067a4f5a268f0e4ec` |
| us-west-2c        | `10.20.2.0/24` | `subnet-09069961189ce1397` |

### Peering Connections
| Peer VPC         | Peer CIDR      | Peering Connection ID   | Bidirectional routing |
|------------------|----------------|-------------------------|-----------------------|
| factom-mainnet-1 | `10.1.0.0/16`  | `pcx-004dd0483fb5746d3` | Yes                   |
| factom-mainnet-2 | `10.0.0.0/16`  | `pcx-05f56dffdb7d0d3d4` | Yes                   |
| factom-testnet   | `10.10.0.0/16` | `pcx-09837944cd069b570` | Yes                   |

### ec2 Instances
| Host Name                      | Type     | IPv4          | Availability Zone | Services                              | Instance ID         |
|--------------------------------|----------|---------------|-------------------|---------------------------------------|---------------------|
| bastion.canonical-ledgers.com  | t2.nano  | `10.20.1.146` | us-west-2b        | OpenSSH with Google Authenticator 2FA | i-080e2469b4ae32fb1 |
| influxdb.canonical-ledgers.com | t2.small | `10.20.0.50`  | us-west-2a        | Update server, InfluxData TICK stack  | i-0b3cb53a13786d810 |

## factom-testnet
The factom-testnet VPC is for running factom nodes on the testnet.

### Details
| Key     | Value                   |
|---------|-------------------------|
| Name    | factom-testnet          |
| VPC ID  | `vpc-04494e34ee7f90cf6` |
| Region  | us-west-2 (Oregon)      |
| CIDR    | `10.10.0.0/16`          |

### Subnets
| Availability Zone | CIDR           | Subnet ID                  |
|-------------------|----------------|----------------------------|
| us-west-2a        | `10.10.0.0/24` | `subnet-005ecbb0707e8fda1` |
| us-west-2b        | `10.10.1.0/24` | `subnet-00a5dfd0863e2edf9` |
| us-west-2c        | `10.10.2.0/24` | `subnet-003d400c602287bb3` |

### Peering Connections
| Peer VPC     | Peer CIDR      | Peering Connection ID   | Bidirectional routing |
|--------------|----------------|-------------------------|-----------------------|
| web-services | `10.20.0.0/16` | `pcx-09837944cd069b570` | Yes                   |

### ec2 Instances
| Host Name                      | IPv4          | Availability Zone | Services                              | Instance ID         |
|-----------------|-------------|-------------------|--------------------------|-------------|
|                 | `10.10.0.0` | us-west-2a        | factomd Auth Node        |             |
|                 | `10.10.1.0` | us-west-2b        | factomd Auth Node Backup |             |
|                 | `10.10.2.0` | us-west-2c        | factomd Follower Node    |             |

## factom-mainnet-1
The factom-mainnet-1 VPC is for running factom nodes on mainnet in the
us-west-1 (N. California) region.

### Details
| Key     | Value                     |
|---------|---------------------------|
| Name    | factom-mainnet-1          |
| VPC ID  | `vpc-0b6a5e1793d2ef319`   |
| Region  | us-west-1 (N. California) |
| CIDR    | `10.1.0.0/16`             |

### Subnets
| Availability Zone | CIDR          | Subnet ID                  |
|-------------------|---------------|----------------------------|
| us-west-1a        | `10.1.0.0/24` | `subnet-0a85b3e84eed377fa` |
| us-west-1b        | `10.1.1.0/24` | `subnet-0ee349ef6c135e1d4` |

### Peering Connections
| Peer VPC         | Peer CIDR      | Peering Connection ID   | Bidirectional routing |
|------------------|----------------|-------------------------|-----------------------|
| web-services     | `10.20.0.0/16` | `pcx-004dd0483fb5746d3` | Yes                   |
| factom-mainnet-2 | `10.0.0.0/16`  | `pcx-089abf97fee0be031` | Yes                   |

### ec2 Instances
| Host Name                       | Type      | IPv4         | Availability Zone | Services            | Instance ID         |
|---------------------------------|-----------|--------------|-------------------|---------------------|---------------------|
| node1.us-west-1.factom-main.net | t2.xlarge | `10.1.0.220` | us-west-1a        | factomd Auth Node   | i-0210512e710a93ca3 |
| node2.us-west-1.factom-main.net | t2.medium | `10.1.1.253` | us-west-1b        | factomd backup Node | i-0488fbc0bbc6bb0ef |

## factom-mainnet-2
The factom-mainnet-2 VPC is for running factom nodes on mainnet in the
us-west-2 (Oregon) region.

### Details
| Key     | Value              |
|---------|--------------------|
| Name    | factom-mainnet-2   |
| VPC ID  | `vpc-43f9dc3a`     |
| Region  | us-west-2 (Oregon) |
| CIDR    | `10.0.0.0/16`      |

### Subnets
| Availability Zone | CIDR          | Subnet ID         |
|-------------------|---------------|-------------------|
| us-west-2a        | `10.0.0.0/24` | `subnet-16088e6f` |
| us-west-2b        | `10.0.1.0/24` | `subnet-0f55f544` |
| us-west-2c        | `10.0.2.0/24` | `subnet-865431dc` |

### Peering Connections
| Peer VPC         | Peer CIDR      | Peering Connection ID   | Bidirectional routing |
|------------------|----------------|-------------------------|-----------------------|
| web-services     | `10.20.0.0/16` | `pcx-05f56dffdb7d0d3d4` | Yes                   |
| factom-mainnet-1 | `10.0.0.0/16`  | `pcx-089abf97fee0be031` | Yes                   |

### ec2 Instances
| Host Name                       | Type      | IPv4         | Availability Zone | Services            | Instance ID         |
|---------------------------------|-----------|--------------|-------------------|---------------------|---------------------|
| node1.us-west-2.factom-main.net | t2.xlarge | `10.0.2.38`  | us-west-2c        | factomd Auth Node   | i-0cf976cd3688bea61 |
| node2.us-west-2.factom-main.net | t2.medium | `10.0.0.200` | us-west-2a        | factomd backup Node | i-0bda0ce4342a06455 |

