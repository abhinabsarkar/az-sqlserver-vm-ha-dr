# Azure SQL Virtual Machines HA and DR configuration design options

SQL Server running on Azure virtual machines have some Azure infrastructure specific nuances when configuring HA & DR. For [AG listener](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-vnn-azure-load-balancer-configure?tabs=ilb) and [FCI](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/failover-cluster-instance-vnn-azure-load-balancer-configure?tabs=ilb) there is a dependency on Azure Load Balancer to get [floating IP](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/failover-cluster-instance-vnn-azure-load-balancer-configure?tabs=ilb) functionality within a single subnet, due to lack of [Address Resolution Protocol (ARP)](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) in public cloud. 

### Option 1: 

Azure Load Balancer introduce additional management and operational overhead for connecting to AG listener and FCI. Azure Load Balancer also induces failover latency of 10 seconds for the load balancing probe (2 unhealthy threshold at 5 second interval being the minimum) to detect the new SQL Server primary. 

### Option 2: 

Using Distributed Network Name (DNN) for AG listener and FCI that avoids the need of having Azure Load Balancer. But DNN does come with following limitations:

* Works only on SQL Server versions starting with either SQL Server 2019 CU8 and later, SQL Server 2017 CU25 and later, or SQL Server 2016 SP3 and later on Windows Server 2016 and later.
* DNN AG listener MUST be configured with a unique port. The port cannot be shared with any other connection on any replica.
* DNN AG listener cannot use SQL Server [default port of 1433](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-distributed-network-name-dnn-listener-configure#port-considerations)
* There are [additional considerations](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-dnn-interoperability) when using DNN with AG.
* In case of FCI, even after creating DNN the original virtual network name (VNN) and virtual IP cannot be deleted as they are necessary components of the FCI infrastructure. In addition, there is [another step](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/failover-cluster-instance-distributed-network-name-dnn-configure#avoid-ip-conflict) needed to prevent the VNN virtual IP address from being assigned to another resource in Azure as a duplicate.
* There are [additional considerations](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/failover-cluster-instance-dnn-interoperability) when using DNN with FCI.

### Option 3: 

Azure SQL VM HA & DR configuration by deploying VMs in multiple subnets and thus eliminate the need for an Azure Load Balancer. Multi subnet configuration helps to match on-premises experience for connecting to your AG listener or FCI. Multi subnet configuration works natively on all supported versions of SQL Server & Windows Server simplifying deployment, maintenance and improving failover time and is Generally Available (GA). 

Reference architecture & [link](https://techcommunity.microsoft.com/t5/azure-sql-blog/simplify-azure-sql-virtual-machines-ha-and-dr-configuration-by/ba-p/2882897)

![alt txt](/images/sqlserver-hadr.png)

## Best practices for High Availability / Disaster Recovery 

1. Deploy your SQL Server VMs to multiple subnets whenever possible to avoid the dependency on an Azure Load Balancer or a distributed network name (DNN) to route traffic to your HADR solution. Refer [link](https://techcommunity.microsoft.com/t5/azure-sql-blog/simplify-azure-sql-virtual-machines-ha-and-dr-configuration-by/ba-p/2882897).

2. Configure [cluster quorum](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/hadr-cluster-best-practices?view=azuresql&tabs=windows2012#quorum-voting) voting to use 3 or more odd number of votes to prevent a split-brain scenario. Don't assign votes to DR regions.  
    a. Although a two-node cluster will function without a quorum resource, it is strictly required to use a quorum resource to have production support. Refer [link](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/hadr-windows-server-failover-cluster-overview?view=azuresql#quorum).  
    b. Quorum option for SQL Server on Azure Virtual Machines (VMs) - a disk witness, a cloud witness, and a file share witness. Refer [link](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/hadr-cluster-quorum-configure-how-to?view=azuresql&tabs=powershell).

Refer [link](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/hadr-cluster-best-practices?view=azuresql&tabs=windows2012#checklist) on the best practices.