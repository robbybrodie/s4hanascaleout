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
- **AWS backplane fencing** using `fence_aws` (no SBD fencing)
- **ANGI SPA resource agent** for HANA resource management
- Majority maker node in AZ3 ensures proper quorum for split-brain prevention

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
    COORD1 -.-> FENCE["fence_aws AWS Backplane"]
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

## Key Features

- **Zero-downtime failover** between primary sites using Pacemaker
- **Synchronous replication** for high availability (AZ1 ↔ AZ2)
- **Asynchronous replication** for disaster recovery (AZ1/AZ2 → AZ3)
- **Scale-out HANA architecture** with coordinator and worker nodes
- **AWS-native fencing** using fence_aws for reliable cluster operations
- **Enterprise storage** with NetApp OnTAP on AWS
- **Dynamic connection routing** via AWS Load Balancers

## Technology Stack

- **Database**: SAP HANA (Scale-out configuration)
- **Clustering**: Pacemaker with ANGI SPA resource agent
- **Fencing**: fence_aws (AWS backplane fencing)
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

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.
