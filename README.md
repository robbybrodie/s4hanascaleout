# S4HANA Scale-Out Architecture on AWS EC2

## Overview

This project documents and implements a highly available SAP S4HANA scale-out architecture deployed across three AWS Availability Zones using Pacemaker clustering for automated failover and high availability management.

## Architecture Description

The architecture implements a sophisticated multi-tier approach combining high availability (HA) and disaster recovery (DR) capabilities:

### High Availability Design (AZ1 & AZ2)
- **Two primary sites** across separate AWS Availability Zones
- Each site contains **two HANA servers** configured as Pacemaker cluster nodes
- **Scale-out configuration**: One coordinator node and one worker node per site
- **Active-passive site design**: Only one site is active at any given time
- **Synchronous database replication** between the two primary sites for zero data loss

### Disaster Recovery Design (AZ3)
- **Third site** in separate AWS Availability Zone for DR purposes
- Contains **majority maker Pacemaker node** for cluster quorum
- Hosts **two additional HANA servers** (non-clustered) for backup purposes
- Receives **asynchronous replication** from whichever primary site is currently active

### Clustering & Fencing
- **Pacemaker clustering** manages automatic failover between sites
- **AWS backplane fencing** using `fence_aws_vpc_net` (no SBD fencing)
- **ANGI SPA resource agent** for HANA resource management
- **Custom VPC isolation resource agent** using NFTables for split-brain prevention
- Majority maker node in AZ3 ensures proper quorum for split-brain prevention
- **Portblock resource agent** provides VPC-level isolation to prevent partial site failures

### Network & Connectivity
- **NetWeaver application servers** connect to database tier
- **AWS Load Balancer** fronts coordinator nodes for connection routing
- Dynamic IP resolution: Applications query load balancer to determine active coordinator, then establish direct connections to appropriate database servers

### Infrastructure Specifications
- **Operating System**: RHEL 8.8 across all servers
- **Compute**: EC2 Superdome instances
- **Storage**: NetApp OnTAP on AWS for enterprise-grade storage
- **Network**: Multi-AZ deployment across three AWS Availability Zones

## Architecture Diagram

```mermaid
flowchart TB
    subgraph AWS["AWS Region"]
        subgraph AZ1["AZ1 - Primary Site A"]
            LB1["AWS Load Balancer AZ1"]
            COORD1["HANA Coordinator<br/>Pacemaker Node<br/>RHEL 8.8"]
            WORKER1["HANA Worker<br/>Pacemaker Node<br/>RHEL 8.8"]
            STORAGE1["OnTAP Storage AZ1"]
        end
        
        subgraph AZ2["AZ2 - Primary Site B"]
            LB2["AWS Load Balancer AZ2"]
            COORD2["HANA Coordinator<br/>Pacemaker Node<br/>RHEL 8.8"]
            WORKER2["HANA Worker<br/>Pacemaker Node<br/>RHEL 8.8"]
            STORAGE2["OnTAP Storage AZ2"]
        end
        
        subgraph AZ3["AZ3 - DR Site"]
            MAJORITY["Majority Maker<br/>Pacemaker Node<br/>RHEL 8.8"]
            BACKUP1["HANA Backup Server 1<br/>Non-clustered<br/>RHEL 8.8"]
            BACKUP2["HANA Backup Server 2<br/>Non-clustered<br/>RHEL 8.8"]
            STORAGE3["OnTAP Storage AZ3"]
        end
        
        subgraph APP["Application Tier"]
            NW1["NetWeaver App Server 1"]
            NW2["NetWeaver App Server 2"]
            NW3["NetWeaver App Server N"]
        end
    end
    
    %% Load balancer connections
    LB1 --> COORD1
    LB2 --> COORD2
    
    %% Scale-out connections within sites
    COORD1 -.-> WORKER1
    COORD2 -.-> WORKER2
    
    %% Storage connections
    COORD1 --> STORAGE1
    WORKER1 --> STORAGE1
    COORD2 --> STORAGE2
    WORKER2 --> STORAGE2
    BACKUP1 --> STORAGE3
    BACKUP2 --> STORAGE3
    
    %% Synchronous replication between primary sites
    COORD1 <==> COORD2
    WORKER1 <==> WORKER2
    
    %% Asynchronous replication to DR site
    COORD1 -.-> BACKUP1
    COORD2 -.-> BACKUP1
    
    %% Pacemaker cluster connections
    COORD1 -.-> MAJORITY
    WORKER1 -.-> MAJORITY
    COORD2 -.-> MAJORITY
    WORKER2 -.-> MAJORITY
    
    %% Application connections
    NW1 --> LB1
    NW1 --> LB2
    NW2 --> LB1
    NW2 --> LB2
    NW3 --> LB1
    NW3 --> LB2
    
    %% AWS Fencing
    COORD1 -.-> FENCE["fence_aws_vpc_net AWS Backplane"]
    COORD2 -.-> FENCE
    WORKER1 -.-> FENCE
    WORKER2 -.-> FENCE
    MAJORITY -.-> FENCE
    
    %% Styling
    classDef primary fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef dr fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef app fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef storage fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef fence fill:#ffebee,stroke:#b71c1c,stroke-width:2px
    
    class COORD1,WORKER1,LB1 primary
    class COORD2,WORKER2,LB2 primary
    class MAJORITY,BACKUP1,BACKUP2 dr
    class NW1,NW2,NW3 app
    class STORAGE1,STORAGE2,STORAGE3 storage
    class FENCE fence
```

## Pacemaker Cluster Architecture

The Pacemaker cluster manages the high availability aspects of the S4HANA deployment with sophisticated resource management and split-brain prevention mechanisms.

### Custom Portblock Resource Agent

A critical innovation in this architecture is the custom **Portblock resource agent** that addresses a fundamental limitation in traditional SAP scale-out HA deployments. This resource agent:

- **Uses NFTables** to provide VPC-level network isolation
- **Prevents partial site failures** that could lead to split-brain scenarios
- **Coordinates both nodes in an AZ** when one node fails or requires maintenance
- **Addresses SAP scale-out limitations** where existing resource agents (including ANGI SPA) cannot alter the state of both nodes simultaneously
- **Eliminates the window** where half a site could continue servicing requests during failover

### Split-Brain Prevention Problem

Traditional SAP scale-out architectures using Pacemaker for HA across two sites suffer from a critical flaw: when one node in a site is taken down (either by fencing or manual maintenance), the existing SAP and ANGI SPA resource agents cannot coordinate the shutdown of both nodes in that AZ. This creates a dangerous window where:

1. One node in the failing site continues to service database requests
2. The other site begins promotion procedures
3. A split-brain scenario occurs until full site promotion completes

**This issue affects every SAP scale-out architecture using Pacemaker for HA across two sites.**

### Portblock Solution

The custom Portblock resource agent solves this by:
- **Immediate VPC isolation** using NFTables rules
- **Coordinated shutdown** of entire AZ when any node fails
- **Prevention of partial site operation** during failover windows
- **Integration with Pacemaker** resource management

```mermaid
graph TD
    subgraph AZ1["AZ1 - Pacemaker Cluster Nodes"]
        COORD1["HANA Coordinator<br/>+ ANGI SPA Agent<br/>+ Portblock Agent"]
        WORKER1["HANA Worker<br/>+ ANGI SPA Agent<br/>+ Portblock Agent"]
    end
    
    subgraph AZ2["AZ2 - Pacemaker Cluster Nodes"]
        COORD2["HANA Coordinator<br/>+ ANGI SPA Agent<br/>+ Portblock Agent"]
        WORKER2["HANA Worker<br/>+ ANGI SPA Agent<br/>+ Portblock Agent"]
    end
    
    subgraph AZ3["AZ3 - Majority Maker"]
        MAJORITY["Majority Maker<br/>Quorum Provider"]
    end
    
    subgraph FENCE["Fencing & Isolation"]
        AWS_FENCE["fence_aws_vpc_net<br/>AWS Backplane"]
        PORTBLOCK["Portblock Agent<br/>NFTables VPC Isolation"]
    end
    
    %% Cluster communication lines
    COORD1 -.-> COORD2
    WORKER1 -.-> WORKER2
    COORD1 -.-> WORKER2
    WORKER1 -.-> COORD2
    
    %% Majority maker quorum
    COORD1 -.-> MAJORITY
    WORKER1 -.-> MAJORITY
    COORD2 -.-> MAJORITY
    WORKER2 -.-> MAJORITY
    
    %% Fencing connections
    COORD1 --> AWS_FENCE
    WORKER1 --> AWS_FENCE
    COORD2 --> AWS_FENCE
    WORKER2 --> AWS_FENCE
    MAJORITY --> AWS_FENCE
    
    %% Portblock isolation
    COORD1 --> PORTBLOCK
    WORKER1 --> PORTBLOCK
    COORD2 --> PORTBLOCK
    WORKER2 --> PORTBLOCK
```

## Key Features

- **Zero-downtime failover** between primary sites using Pacemaker
- **Synchronous replication** for high availability (AZ1 ↔ AZ2)
- **Asynchronous replication** for disaster recovery (AZ1/AZ2 → AZ3)
- **Scale-out HANA architecture** with coordinator and worker nodes
- **AWS-native fencing** using fence_aws_vpc_net for reliable cluster operations
- **Enterprise storage** with NetApp OnTAP on AWS
- **Dynamic connection routing** via AWS Load Balancers
- **Custom Portblock resource agent** for VPC-level isolation and split-brain prevention

## Technology Stack

- **Database**: SAP HANA (Scale-out configuration)
- **Clustering**: Pacemaker with ANGI SPA resource agent
- **Fencing**: fence_aws_vpc_net (AWS backplane fencing)
- **Custom Resource Agent**: Portblock (NFTables-based VPC isolation)
- **Operating System**: Red Hat Enterprise Linux 8.8
- **Compute**: AWS EC2 Superdome instances
- **Storage**: NetApp OnTAP on AWS
- **Load Balancing**: AWS Application Load Balancer
- **Replication**: HANA System Replication (HSR)

## Implementation Notes

- Sites are designed for active-passive operation (only one site active at a time)
- Majority maker in AZ3 prevents split-brain scenarios
- NetWeaver applications use load balancer for initial coordinator discovery
- Direct database connections established after coordinator IP resolution
- Backup servers in AZ3 provide additional recovery options
- **Portblock resource agent addresses universal SAP scale-out HA limitation** affecting all two-site Pacemaker deployments
- Custom NFTables rules provide immediate VPC isolation during node failures
- Coordinated site-level shutdown prevents partial site operation during failover

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the [Apache 2.0 License](LICENSE) - see the LICENSE file for details.
