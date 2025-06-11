---

copyright:
  years: 2025
lastupdated: "2025-06-11"
keywords: cyber resiliency, ransomware,  business continuity, backup and restore
subcollection: pattern-cyber-resiliency-vpc

authors:
  - name: Ana Biazetti
  - name: Balaji Viswanathan

# The release that the reference architecture describes
version: 1.0

# Use if the reference architecture has deployable code.
# Value is the URL to land the user in the IBM Cloud catalog details page for the deployable architecture.
# See https://test.cloud.ibm.com/docs/get-coding?topic=get-coding-deploy-button
# deployment-url: url
use-case: BackupAndRestore, BusinessContinuity, CyberResilience, Cyberattack, Cybersecurity, IncidentResponse, Ransomware
compliance: IBMCloudFFS
docs: /docs/pattern-cyber-resiliency-vpc

content-type: reference-architecture

---


{{site.data.keyword.attribute-definition-list}}



# Cyber resiliency pattern on VPC
{: #cyber-resiliency}
{: toc-content-type="reference-architecture"}
{: toc-version="1.0"}
{: toc-use-case="BackupAndRestore,BusinessContinuity,CyberResilience,Cyberattack,Cybersecurity,IncidentResponse,Ransomware"}
{: toc-compliance="IBMCloudFFS"}





The cyber resiliency reference architecture provides an overview and details for designing a secure cyber recovery solution on Virtual Private Cloud (VPC). Ransomware attacks attempt to encrypt, exfiltrate, or otherwise render primary and backup copies of data and configuration inoperable. The key objective of cyber recovery is on protecting, by backing up the workloads to a secure data bunker and validating available recovery points in an isolated cleanroom environment. Finally, in the event of a ransomware attack, recovering valid recovery points to a new and clean recovery environment. Unlike backup and DR solutions where the primary objective might be low RPO, the focus here is on clean recovery and returning business to working order swiftly and effectively. 

The reference architecture is built on the VSI on VPC secure landing zone architecture and incorporates many of the cyber resiliency principles that are outlined in the [Well Architected Framework](https://www.ibm.com/architectures/well-architected/resiliency).

## Architecture diagram
{: #architecture-diagram }


![High level architecture.](images/cyber-resiliency-hld.svg "Cyber Resiliency High level architecture"){: caption="Cyber Resiliency high level architecture" caption-side="bottom"}

The high-level architecture diagram gives an overview of the different environments and components that constitute the solution. It provides a generic overview of the requirements that drive the architectural decisions. 

- The source environment with workloads to protect might be on premises or on a public cloud. Similarly, the recovery environment where the workloads and are restored, might be on premises or on the same or different public cloud. For the rest of this reference architecture, we consider IBM Cloud VPC as both the source and recovery environment.
- Management VPC hosts the secure access components - VPN Gateway and Bastion Host. External clients connect to the cyber resiliency environment through the management VPC. All access is logged for security audit. Identity and access management enforces restricted access only to cyber admins team to this environment. 
- DataMover VPC is a virtually air-gapped environment that hosts the data mover component and backs up data from source environment by using a pull mechanism and stores the snapshots in an immutable storage.
- Cleanroom VPC is an isolated environment that hosts tools to analyze the backup snapshots, restore them, and identify clean recovery points. It can be torn down and rebuilt as needed.
- A recovery environment running on a separate VPC hosts the minimal set of workloads (and data) for business continuity in the event of a cyberattack. When needed, this environment is built as a clean infrastructure into which a valid and verified recovery point is restored and endpoints exposed in-lieu of source environment.

The following detailed reference architecture expands the high-level diagram for a single zone deployment. For a multi-zone deployment, use principles laid out in [multi-zone resiliency patterns](/docs/pattern-vpc-vsi-multizone-resiliency?topic=pattern-vpc-vsi-multizone-resiliency-resiliency-design)
![Detailed reference architecture.](images/cyber-resiliency-reference-architecture.svg "Cyber Resiliency reference architecture on VPC"){: caption="Cyber Resiliency reference architecture" caption-side="bottom"}

- The production VPC (which maps to the source environment discussed earlier) shows a nonexhaustive list of workloads that need to be protected. It might be in a separate IBM Cloud account within the enterprise account structure.
- The choice of the Data mover component, which does the backup and restore is left open. The architecture allows for different products with different deployment models, workload support, and functional capabilities.
- Transit Gateway connects DataMover VPC and Production VPC, DataMover VPC and Recovery VPC, Management VPC and other VPCs. 
- Virtual Air-Gap is enforced through VPC Access Control List and Security Groups, alternatively, a network firewall appliance might provide the functions.
- All VPC resources use Virtual Private Endpoints to connect to Cloud services, ensuring that all traffic remains on the private backbone.
- The Air-Gap can be extended to Cloud Services, like Cloud Object Storage by using Context Based Restrictions (CBRs) to allow only access from the Data Mover VPC to the direct endpoints of the service instances.
- Cloud Object Storage buckets are created with versioning and object lock feature that is enabled to provide Write-Once-Read-Many (WORM) capability. Multiple buckets with policies that match those governing data retention and immutable storage requirements for workload backups can be created.

## The Flow
{: #flow}

The detailed architecture diagram shows the various steps that are followed to backup, verify, and recover workloads. Initially, using secure access through the bastion host, the data mover components are deployed and configured to protect workloads in source. By default, the virtual air-gap is closed with rules to deny external access to data mover resources.

1. At scheduled intervals, as a pre-step to the backup task, the virtual air-gap is opened by applying rules to allow traffic in and out of data mover VPC to selective hosts or subnets in the source environment.
2. The Data Mover component runs the backup task, often with the help of agents that are installed on workload endpoints to pull data into local storage. The local storage might act as a cache and holds the data before it is moved to permanent storage.
3. Post completion of the backup task, the virtual air-gap is closed by applying rules to deny all but necessary management traffic.
4. The backup snapshots are written to Cloud Object Storage buckets as either primary or secondary immutable copy of the data. This operation might be asynchronous or synchronous with backup task, depending on its SLA.
5. A cleanroom VPC is provisioned to verify the latest snapshot for ransomware or malware signatures. It is a clean and isolated environment and can be destroyed partially or fully post verification.
6. Verification of backups can be offline or online. A mount server with forensic tools that analyzes the offline backup for ransomware signatures or encryption patterns or entropy scores might be faster when data sizes are large. In some cases, the snapshot is restored into a live instance, the application that is deployed and tested to verify the backup. Successfully verified backups of related workloads represent a clean logical recovery point group to be restored together.
7. When a cyberattack is detected on the source environment, automation tools are used to provision a clean recovery environment. The recovery environment might be identical to the source environment or be a subset necessary to bring up the minimal set of workloads necessary.
8. A clean recovery point group is selected and restored to bring up the workloads and expose them in place of the infected source environment.

The new recovery environment now becomes the source and data mover component is re-configured to start protecting these workloads. Forensic analysis and cleaning of the original source environment when completed might trigger a restore back to it or the recovery environment itself might be promoted as a new production environment.

## Design concepts
{: #design-concepts}

![Enter image alt text here.](images/cyber-resiliency-heat-map.svg "Cyber Resiliency design requirements"){: caption="Cyber resiliency design requirements" caption-side="bottom"}

The {{site.data.keyword.arch_framework}} provides a consistent approach to design cloud solutions by addressing requirements across a set of "aspects" and "domains", which are technology-agnostic architectural areas that need to be considered for any enterprise solution. See [Introduction](/docs/architecture-framework) to the {{site.data.keyword.arch_framework}} for more details.

## Requirements
{: #requirements}

The following table outlines the requirements that are addressed in this architecture.

| Aspect | Requirements |
| -------------- | -------------- |
| Compute            | <ul><li>Provide properly isolated compute resources with adequate compute capacity for the applications.</li></ul> |
| Storage            | <ul><li>Provide storage that meets the application and database performance requirements.</li></ul> |
| Networking         | <ul><li>Deploy workloads in an isolated environment and enforce information flow policies.</li><li>Provide secure, encrypted connectivity to the cloud’s private network for management purposes.</li><li>Distribute incoming application requests across available compute resources.</li><li>Support failover of application to an alternative site in the event of planned or unplanned outages.</li><li>Provide public and private DNS resolution to support the use of hostnames instead of IP addresses.</li></ul> |
| Security           | <ul><li>Ensure that all operator actions are executed securely through a bastion host.</li><li>Protect the boundaries of the application against denial-of-service and application-layer attacks.</li><li>Encrypt all application data in transit and at rest to protect it from unauthorized disclosure.</li><li>Encrypt all backup data to protect it from unauthorized disclosure.</li><li>Encrypt all security data (operational and audit logs) to protect from unauthorized disclosure.</li><li>Encrypt all data by using customer-managed keys to meet regulatory compliance requirements for additional security and customer control.</li><li> Protect secrets through their entire lifecycle and secure them using access control measures.</li></ul> |
| Resiliency         | <ul><li>Support application availability targets and business continuity policies.</li><li>Ensure availability of the application in the event of planned and unplanned outages.</li><li>Provide highly available compute, storage, network, and other cloud services to handle application load and performance requirements.</li><li>Backup application data to enable recovery in the event of unplanned outages.</li><li>Provide highly available storage for security data (logs) and backup data.</li><li>Automate recovery tasks to minimize downtime</li></ul> |
| Service Management | <ul><li>Monitor system and application health metrics and logs to detect issues that might impact the availability of the application.</li><li>Generate alerts/notifications about issues that might impact the availability of applications to trigger appropriate responses to minimize downtime.</li><li>Monitor audit logs to track changes and detect potential security problems.</li><li> Provide a mechanism to identify and send notifications about issues that are found in audit logs.</li><li>Day-0 automation for consistent and secure infrastructure provisioning and Day-2 operations automation for ease of management</li></ul> |
{: caption="Requirements" caption-side="bottom"}

## Components
{: #components}

The following table outlines the products or services that are used in the architecture for each aspect.

| Aspects | Architecture components | How the component is used |
| -------------- | -------------- | -------------- |
| Compute | [VPC VSIs](/docs/vpc?topic=vpc-creating-virtual-servers&interface=ui) | Data mover components are deployed on a cluster of VSIs to provide scale and high availability |
| Storage | [VPC Block Storage](/docs/vpc?topic=vpc-creating-block-storage&interface=ui) | Block Storage for VPC VSI |
|  | [Cloud Object Storage](/docs/cloud-object-storage?topic=cloud-object-storage-about-cloud-object-storage) | Cloud Object Storage Buckets with Object Lock enabled |
| Networking | [VPC Virtual Private Network (VPN)](/docs/iaas-vpn?topic=iaas-vpn-getting-started) | Remote access to manage resources in a private network |
|  | [Transit Gateway](/docs/transit-gateway?topic=transit-gateway-about) |  Connects across VPCs and source and recovery environments |
|  | [Virtual Private Gateway & Virtual Private Endpoint (VPE)](/docs/vpc?topic=vpc-about-vpe) | For private network access to Cloud Services, for example, Key Protect, Cloud Object Storage, and so on. |
|  | [Public Gateway](/docs/vpc?topic=vpc-about-public-gateways) | For resource access to the internet |
| Security | [IAM](/docs/account?topic=account-cloudaccess) | IBM Cloud Identity & Access Management |
|  | [BYO Bastion Host on VPC VSI](/docs/framework-financial-services?topic=framework-financial-services-vpc-architecture-connectivity-bastion-tutorial-teleport) | Remote access with Privileged Access Management |
|  | [VPCs, Subnets, Security Groups and ACLS](/docs/vpc?topic=vpc-about-subnets-vpc&interface=ui) |  Network isolation and Virtual Air-Gap |
|  | [Key protect or HPCS](/docs/overview?topic=overview-key-encryption) | Hardware security module (HSM) and Key Management Service |
|  | [Secrets Manager](/docs/secrets-manager?topic=secrets-manager-getting-started) | Certificate and Secrets Management |
|  | [Context Based Restrictions](/docs/vpc?topic=vpc-cbr&interface=ui) |  Enforce access restrictions for service instances based on a rule's criteria |
|  Delivery pipeline | [Toolchain](/docs/devsecops?topic=devsecops-devsecops_intro) | Pipeline for build and deploy |
|  Resiliency | Cloud Object Storage | Data archived in Cloud Object Storage cross-region buckets |
| Service Management | [IBM Cloud Monitoring](/docs/monitoring?topic=monitoring-about-monitor) | Apps and operational monitoring |
|  | [IBM Cloud Logs](/docs/cloud-logs?topic=cloud-logs-getting-started) | Apps and operational logs |
|  | [Activity Tracker Event Routing](/docs/atracker?topic=atracker-about) | Audit logs |
|  | [Schematics](/docs/schematics?topic=schematics-learn-about-schematics) | Automated deployment that uses Deployable Architecture and Ansible actions |
{: caption="Components" caption-side="bottom"}
