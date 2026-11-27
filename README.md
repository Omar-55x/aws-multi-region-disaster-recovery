# Solution Overview
The solution demonstrates a multi-region disaster recovery strategy for a web application with Route 53 failover. A primary region (us-east-1) runs the primary stack. A secondary region (eu-west-1) maintains warm standby resources. Route 53 health checks detect primary region failure and automatically failover DNS. Failover actions automation is implemented by Lambda functions that are triggered by EventBridge on failover. Backups provide an additional layer of data protection against scenarios that replication cannot handle. Primary region's infrastructure will be restored and data will be synchronized to failback to the primary region.

The selected DR strategy for this solution is the **Warm Standby** strategy.

# Architecture Diagrams

The architecture is split into two diagrams:

## Main Architecture
![Architecture](./architecture/architecture-diagram.png)
*Primary region (us-east-1) containts the primary resources. Second region (eu-west-1) contains scaled-down enviornment*

## Failover Automation Flow
![FailoverFlow](./architecture/failover-automation-flow.png)

# Architecture Componenets
- **DNS & Routing:** Domains are managed by **Route 53** as the DNS and traffic manager using a failover active-passive routing policy.
- **Compute:** Provided using **EC2** instances for compute power, **Auto Scaling** for handling traffic changes and spikes, and **ALB** for HTTP/S path-based routing.
- **Database:** **Aurora Global Database** is configured to provide cross-region database replication. **DynamoDB Global Tables** provide active-active data replication for low-latency reads and writes across the two regions.
- **Storage:** **S3** stores application files, user-uploaded content, and static assets. **Cross-Region Replication** is configured to allow file accessibility when the primary region is unavailable.
- **Backup:** **AWS Backup** allows restoring data to an earlier recovery point and adding another recovery source with centralized backup plans for RDS, EC2, and S3.
- **Security:** **IAM** is used to manage authentication and access to resources using permissions and different types of policies. Data is encrypted at rest with **KMS** using AWS-managed keys.
- **Monitoring:** Application metrics and alarms are managed through **CloudWatch** to monitor health of the resources and make sure the application is running smoothly.
- **Automation:** **EventBridge** and **Lambda** are used to handle the recovery actions that should be taken during a failover.
- **Infrastructure:** **CloudFormation StackSets** are used to handle the creation and deployment of the application infrastructure across the two regions.

# Disaster Recovery Strategies
Having a look at the four main disaster recovery strategies:  
1. **Backup & Restore** 
   - Consists of taking regular backups stored in a cost-optimized storage. When a disaster or a failure occurs, data is pulled back from S3 in a DR region. If on-premises, data can be downloaded from S3 to on-premises servers.
   - AWS Backup is typically used to automate backups across services form a central place.
   - Offers the most cost-effective DR solution, but the highest RTO/RPO.
2. **Pilot Light**
   - This strategy keeps a minimal, always-on version of your environment in a secondary region.
   - Typically, the database is actively running and replicating data while other infrastructure and pre-configured but not running.
   - When a disaster occurs, compute is scaled up and traffic is redirected to the DR region.
   - Recovery is not instant, but offers a lower RTO/RPO compated to the Backup & Restore strategy.
3. **Warm Standby**
   - This strategy runs a scaled-down clone of your production environment in the DR region at all times.
   - All the infrastructure is running, unlike Pilot Light, but is scaled down.
   - On failover, you scale out the standby environment to full production size and shift traffic.
   - This strategy offers a lower RTO/RPO than Pilot Light.
4. **Multi-Site Active-Active**
   - This strategy runs two identical, production-scale environments simultaneously in different regions, both actively serving traffic.
   - If one Region fails, the other absorbs 100% of traffic with no downtime.
   - Offers the lowest RTO/RPO of near zero, but incurs the highest costs  
  
### RTO/RPO and Cost Comparison:
| Strategy | Backup & Restore | Pilot Light | Warm Standby | Multi-Site |
|:-------------|:--------------:|:--------------:|:--------------:|:--------------:|
| **RTO/RPO**         |    Hours      |         Tens of minutes |    Minutes      |         Near-zero |
| **Cost**         |     Lowest       |          Low |    Medium      |         Highest |

### Why Warm Standby?
This application is running production workloads that contains critical user data that needs to be kept safe and available at any time. Keeping that in consideration and for optimizing costs, the optimal solution for this case was pointing towards the Warm Standby DR patten.